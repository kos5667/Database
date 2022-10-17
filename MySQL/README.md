## MySQL - performance improvement

> 목차
>
> [Explain](#Explain) : 쿼리 명령문을 어떻게 수행할 것인지 실행 계획에 대한 정보
>
> [Profiles](#Profiles) : 현재 세션 과정 동안 실행 문에 대한 리소스 정보
>
> [Optimizer_trace](#Optimizer_trace) : 실제 실행 시 바람직한 비용 기반으로 계획을 선택한 과정 정보

---

### Explain

**id** 

실행계획의 순서. 이 순서대로 select 문이 실행.



**select_type**

- SIMPLE : 단순 select문
- PRIMARY : 첫번째 쿼리
- DERIVED : select문으로 추출된 테이블 ( from 절에서의 서브쿼리 또는 inline view)
- SUBQUERY : sub query 중 첫번째 select문
- UNION : UNION쿼리에서 PRIMARY를 제외한 나머지 select문
- DEPENDENT SUBQUERY
- DEPENDENT UNION



**table**

대상이 되는 테이블명 혹은 Alias



**partitions**

파티션 사용 시, 대상이 되는 파티션



**type**

data access 타입. 우수한 순서대로 아래 설명. 뒤로 갈수록 나쁜 형태.

- system : 0개 또는 하나의 row를 가진 테이블. const 타입의 특별한 케이스. (MyISAM, Memory 테이블)

- const : primary key나 unique Key의 모든 컬럼에 대해 equal 조건으로 검색 반드시 1건의 레코드만 반환

  ```mysql
  SELECT * FROM tbl_name WHERE primary_key = 1;
  ```

- eq_ref : 조인에서 첫 번째 읽은 테이블의 컬럼값을 이용해 두 번째 테이블을 primary key나 unique Key로 equal 조건 검색으로 두번째 테이블은 반드시 1건의 레코드만 반환 (1:1 관계)

  ```mysql
  SELECT * FROM ref_table, other_table
  WHERE ref_table.key_column = other_table.column;
  ```

- ref : 조인의 순서와 인덱스의 종류와 관계없이 equal 조건으로 검색 (1:n 관계)

  ```mysql
  SELECT * FROM ref_table WHERE key_column = expr;
  ```

- unique_subquery : IN(sub-query) 형태의 조건에서 반환 값에 중복 없음

- index_subquery : unique_subquery와 비슷하지만 반환 값에 중복 있음

- range : 인덱스를 하나의 값이 아니라 범위로 검색. 가장 많이 사용

  ```mysql
  SELECT * FROM tbl_name
  WHERE key_column BETWEEN 0 AND 20;
  ```

- Index : 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔

- All : 풀스캔. 성능 가장 안 좋음



**possible_keys**

해당 테이블에서 데이터를 찾기위해 선택한 인덱스 목록



**key**

실제로 쿼리 실행에 사용한 인덱스



**key_len**

쿼리를 처리하기 위해 단일, 다중 컬럼으로 구성된 인덱스의 각 레코드에서 몇 바이트까지 사용했는지



**ref**

행을 추출하는데 키와 함께 사용된 컬럼이나 상수 값



**rows**

쿼리 수행에서 예상하는 검색해야 할 행수. 조인문이나 서브쿼리 최적화에 있어서 중요한 항목.

조회 결과 수와 rows가 차이가 크다면 성능 개선 필요



**Extra**

쿼리에 관한 추가적인 정보

- distinct : 이미 처리한 값과 동일한 값을 가진 Row는 처리하지 않음.
- not exist : left join을 수행함에 매치되는 한 행을 찾으면 더 이상 매치되는 행을 검색하지 않음.
- Range checked for each record :사용할 좋은 인덱스가 없음.
- using filesort : 정렬을 위해 추가적인 과정을 필요로 함(물리적인 정렬작업 수행)
- using index : 실제 데이터 Block을 읽지 않고 인덱스 Block 만으로 결과를 생성할 수 있는 경우
- using temporary : 임시 테이블을 사용. order by 나 group by 절이 각기 다른 컬럼을 사용할 때 발생
- using where : where절이 다음 조인에 사용될 행이나 클라이언트에게 돌려질 행을 제한하는 경우
- using index for group-by

> 쿼리를 가능한 한 빠르게 하려면, Extra 값의 Using filesort나 Using temporary에 주의해야 함.
>
>  EXPLAIN의 출력 내용 중 rows 컬럼값들을 곱해봄으로써 얼마나 효과적인 join을 실행하고 있는지 알 수 있다.



### Profiles

쿼리가 처리되는 동안 각 단계별 작업에 시간이 얼마나 걸리는지 확인



1. 확인 (Default : 0 (off) / 1 : ON)

   ```mysql
   SELECT @@profiling;
   ```

2. 설정

   ```mysql
   SET profiling = 1;
   ```

3. profiling history size 변경 (Default : 15 / 0 ~ 100)

   ```mysql
   SET @@profiling_history_size = 100;
   ```

4. Query 실행

5. Profile 된 리스트 조회

   ```mysql
   SHOW PROFILES;
   ```

   ![]()

   ```mysql
   SHOW PROFILE ALL FOR QUERY 43; -- ALL 말고도 CPU/IPC/SOURCE/SWAPS 등으로 조회 가능
   ```

   ![]()

   ```mysql
   SELECT STATE, FORMAT(DURATION, 6) AS DURATION
   FROM INFORMATION_SCHEMA.PROFILING
   WHERE QUERY_ID = 43
   ORDER BY SEQ;
   ```

6. 비성화

   ```mysql
   SET profiling = 0;
   ```

   

### Optimizer_trace

Optimizer는 여러 조건들을 통해 쿼리 플랜을 최초 작성하고, 마지막으로 한번 더 정리하는 과정.

테이블에 인덱스가 있어도 비용에 따라 table scan을 사용할 수 있으며 이 경우 인덱스가 사용되지 않은 이유를 확인 가능.

성능 저하가 될 수 있기에 사용 후 off.

1. Optimizer_trace 활성화

   ```mysql
   SHOW VARIABLES LIKE 'optimizer_trace';
   SET OPTIMIZER_TRACE = 'enabled=on';
   ```

   

2. 쿼리 실행

3. information_schema.optimizer_trace 확인

   ```mysql
   SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
   ```

   

4. Optimizer_trace 비활성화

   ```mysql
   SET OPRIMIZER_TRACE = 'enable=OFF';
   ```

   