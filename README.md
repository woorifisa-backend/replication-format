# 📊 MySQL 복제 포맷(SBR vs RBR) 성능 비교

## 개요
MySQL의 `binlog_format` 설정에 따른 복제 메커니즘의 차이를 이해하고,  
실제 운영 환경에서 발생할 수 있는 **데이터 불일치** 및 **복제 지연(Replication Lag)** 현상을  
실험을 통해 분석합니다.

## 실습 결과 요약

## 1️⃣ 데이터 정확성 비교

### 실험 목적

- `UUID()`는 실행할 때마다 다른 값을 생성하는 **비결정 함수**이다.
- SBR은 SQL 문을 복제하는 방식이기 때문에,
  Replica에서 `UUID()`가 다시 실행될 수 있다.
- 그 결과, Source와 Replica 간 값이 달라질 가능성이 있다.

---

## 🛠 테스트 환경 구성

### 1. 실험용 데이터베이스 생성

```sql
CREATE DATABASE IF NOT EXISTS rep_test;
USE rep_test;
```

### 2. 테스트 테이블 생성
```sql
CREATE TABLE t_uuid (
  id INT PRIMARY KEY,
  v VARCHAR(36)
);

INSERT INTO t_uuid (id, v) VALUES
  (1, NULL),
  (2, NULL),
  (3, NULL),
  (4, NULL),
  (5, NULL);
```
- id: 기본 키
- v: UUID 저장용 컬럼 (VARCHAR(36))
- UUID()는 보통 36자 문자열로 생성된다.

---

### 1️⃣ SBR (Statement-Based Replication)
Step 1: 세션에서 binlog 포맷을 SBR로 변경
```sql
SET SESSION binlog_format = 'STATEMENT';
```

Step 2: UUID 값으로 업데이트
```sql
UPDATE rep_test.t_uuid SET v = UUID();
```

### 🔎 동작 원리

- 각 row마다 `UUID()`가 호출되어 `v` 값이 채워진다.
- SBR에서는 binlog에 **최종 결과값이 아니라 SQL 문 자체가 기록**된다.
- Replica는 해당 SQL 문을 다시 실행한다.
- 이때 `UUID()`가 Replica에서 다시 평가될 수 있다.

<img width="384" height="182" alt="스크린샷 2026-02-23 오후 4 44 06" src="https://github.com/user-attachments/assets/158cb6d3-ea95-4912-9b60-b0b9117d9c45" />
<img width="334" height="182" alt="스크린샷 2026-02-23 오후 4 44 26" src="https://github.com/user-attachments/assets/c27dbeea-c02c-43dc-8710-29cb539c11b5" />

> `rep_test.t_uuid` 테이블에서 `v = UUID()`가  
> Source와 Replica에서 각각 독립적으로 실행될 수 있다.

### ✅ 실험 결과
Source와 Replica의 UUID 값이 서로 다르게 나타날 수 있다.

---

### 2️⃣ RBR (Row-Based Replication)
Step 1: 세션에서 binlog 포맷을 RBR로 변경
```sql
SET SESSION binlog_format = 'ROW';
```

Step 2: UUID 값으로 업데이트
```sql
UPDATE rep_test.t_uuid SET v = UUID();
```

### 🔎 동작 원리

- 각 row마다 `UUID()`가 호출되어 `v` 값이 채워진다.
- RBR에서는 SQL 문이 아니라 **변경된 row의 최종 결과 값이 binlog에 기록**된다.
- 즉, `id=1`의 `v`가 `NULL → 특정 UUID 값`으로 변경되었다는 정보가 기록된다.
- Replica는 해당 row 변경 내용을 그대로 적용한다.
- Replica는 `UUID()`를 다시 실행하지 않는다.

<img width="381" height="177" alt="스크린샷 2026-02-23 오후 5 03 23" src="https://github.com/user-attachments/assets/6b157f15-43ce-4752-9f1d-195168e33e9e" />
<img width="327" height="179" alt="스크린샷 2026-02-23 오후 5 04 14" src="https://github.com/user-attachments/assets/9d4e2ce7-1e6d-4cdd-bed1-718d4f4fa26c" />

> Replica는 SQL을 재실행하는 것이 아니라  
> Source에서 이미 결정된 최종 값을 **그대로 복제**한다.

### 실험 결과
Source에서 생성된 UUID 값과 Replica에 적용된 UUID 값이 완전히 동일하게 나타났다.

---

### 결론

SBR은 SQL 문을 그대로 기록하고 Replica에서 다시 실행하는 방식이기 때문에,  
`UUID()`와 같은 비결정 함수가 포함된 경우 Source와 Replica 간 데이터 불일치가 발생할 수 있다.  
즉, 동일한 쿼리를 실행하더라도 동일한 결과가 보장되지는 않는다.

반면 RBR은 변경된 row의 최종 결과 값을 그대로 기록하고 적용하는 방식이므로,  
비결정 함수가 포함되어도 Source와 Replica의 데이터가 항상 동일하게 유지된다.

복제에서 중요한 것은  
“같은 쿼리를 실행하는 것”이 아니라  
“같은 상태를 보장하는 것”이다.

따라서 데이터 정확성과 일관성이 중요한 환경에서는 RBR 방식이 더 적합하다.

---


## 2️⃣ 로그 파일 비교


### 📌 실험 목적

MySQL의 Binlog Format에 따른 로그 크기 차이를 비교하기 위해  
SBR(Statement-Based Replication)과 RBR(Row-Based Replication) 모드에서  
동일한 대량 UPDATE 작업을 수행하고 로그 파일 크기를 비교하였다.

---

### 🧪 실험 환경

- MySQL 8.x
- Binlog 활성화 상태
- 동일한 테이블 및 동일한 UPDATE 조건

---

### 1️⃣ SBR (Statement-Based Replication) 실험

### ① Binlog Format 변경

```sql
SET GLOBAL binlog_format = 'STATEMENT';
```
<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/a56edcc1-a53e-4e27-b102-b78f2b339738" />

---

### ② 테스트 테이블 생성

```sql
DROP TABLE IF EXISTS big_table;

CREATE TABLE big_table (
    id INT PRIMARY KEY AUTO_INCREMENT,
    value INT
);
```
<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/be8a32e0-b97f-406c-ac80-3d531d038484" />

---

### ③ 3647개 더미 데이터 생성

```sql
INSERT INTO big_table(value)
SELECT 1 FROM information_schema.columns LIMIT 10000;
```

→ 실제 삽입 행 수: **3647 rows**

---

<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/efd107a9-ee94-4762-9b8c-af64c7aded38" />


### ④ 로그 구간 분리

```sql
FLUSH LOGS;
```

<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/40c3184d-cc37-4265-9dd8-8bdb5e69dc19" />


→ 새로운 binlog 파일 생성  
→ 이후 실행되는 쿼리만 해당 파일에 기록됨

---

### ⑤ 대량 UPDATE 실행

```sql
UPDATE big_table SET value = value + 1;
```

---

<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/c49f46dc-ee79-4e6d-87fd-6d30d2c7dbff" />


### ⑥ Binlog 파일 크기

- 약 **1.3 KB**

---

### 📖 SBR 기록 방식

- 실행된 **SQL 문 자체를 그대로 기록**
- 예:

```sql
UPDATE big_table SET value = value + 1;
```

대량 변경이어도 SQL 한 줄만 기록되므로 로그 크기가 작다.

---

### 2️⃣ RBR (Row-Based Replication) 실험

### ① Binlog Format 변경

```sql
SET GLOBAL binlog_format = 'ROW';
```

<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/5d3f7caa-720a-4605-96f8-79f646b87a08" />

---

### ② 로그 구간 분리

```sql
FLUSH LOGS;
```

---

### ③ 동일한 대량 UPDATE 실행

```sql
UPDATE big_table SET value = value + 1;
```
<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/de412e1f-c517-4ebf-b89e-bfdd5843d82a" />

---

### ④ Binlog 파일 크기

- 약 **65 KB**

---
<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/feaf7d44-b89c-453e-b664-cd4ef3cfa694" />

### 📖 RBR 기록 방식

- 변경된 **각 row의 실제 데이터 값**을 기록
- 3647개 행 각각에 대한 변경 내용이 저장됨

즉, 변경된 데이터 수만큼 로그가 증가한다.

---

### 3️⃣ Binlog 파일 확인

```sql
SHOW MASTER STATUS;
```

예시 파일:

- SBR → `mysql-bin.000004`
- RBR → `mysql-bin.000006`

파일을 복사하여 내용을 직접 비교하였다.
<img width="302" height="220" alt="image" src="https://github.com/user-attachments/assets/cafb2286-160d-4f3a-b4a0-edc17ad022c1" />

---
<img width="602" height="642" alt="image" src="https://github.com/user-attachments/assets/020f5ea9-44f7-417f-84f5-65e487373a27" />

SBR(004파일)

<img width="597" height="646" alt="image" src="https://github.com/user-attachments/assets/e52acd6c-226d-45e2-9f1b-4aa0c81da614" />


RBR(006파일)


### 📊 SBR vs RBR 비교

| 구분 | SBR | RBR |
|------|------|------|
| 기록 방식 | SQL 문 자체 기록 | 변경된 Row 데이터 기록 |
| 로그 크기 | 약 1.3KB | 약 65KB |
| 대량 UPDATE 영향 | 거의 없음 | 변경된 행 수만큼 증가 |
| 장점 | 로그 크기 작음 | 데이터 정합성 높음 |
| 단점 | 비결정적 쿼리 문제 가능 | 로그 크기 증가 |

---

# 🔥 결론

- **SBR은 실행된 SQL을 기록**
- **RBR은 변경된 데이터 자체를 기록**

따라서,

> SBR은 SQL 한 줄만 기록되지만,  
> RBR은 변경된 모든 row 데이터를 기록하므로  
> 대량 변경 작업에서는 RBR이 훨씬 많은 로그를 생성한다.

이 실험을 통해 Binlog Format에 따른 로그 증가 차이를 직접 확인하였다.





---

## 3️⃣ 실행 시간 비교

### 1. 테스트 환경 및 조건
- **대상 테이블:** `time_test` (약 20,000건의 레코드)  
- **부하 쿼리:**  
  ```sql
  UPDATE ... SET ... WHERE SLEEP(0.0005) = 0;
  ```
- **소스 서버 실행 시간:** 약 **17.68초**  
- **측정 지표:** `SHOW REPLICA STATUS` 내 `Seconds_Behind_Source`

---

### 2. 포맷별 지연 시간(Latency) 비교

#### 🔹 [Case 1] SBR (Statement-Based Replication)
- **지연 시간 추이:** `19s → 37s → 0s`

**현상 분석**
- 소스 서버에서 쿼리가 완료되어 로그가 전달된 시점에 이미 약 **19초의 지연** 발생  
- 레플리카가 전달받은 SQL을 실행하는 동안 실제 시간이 계속 흐르며 **지연 수치가 37초까지 증가**  
- 레플리카가 소스와 동일한 연산(`SLEEP`)을 모두 마친 후에야 지연이 `0s`로 수렴  

**결론:**  
> 소스의 연산 부하가 레플리카에 그대로 전이되어 **복제 지연(Lag)** 이 심화됨

---

#### 🔹 [Case 2] RBR (Row-Based Replication)
- **지연 시간 추이:** `19s → 1~2s → 0s (혹은 즉시 0s)`

**현상 분석**
- 소스에서 쿼리가 끝난 시점에 발생한 **기본 지연(약 19초)** 은 동일  
- 하지만 레플리카는 연산 과정을 생략하고 **최종 변경된 값만 즉시 반영**  
- 쿼리 수신 직후 지연 시간이 급격히 감소  

**결론:**  
> 연산량이 많은 작업에서는 **RBR이 SBR보다 복제 성능과 동기화 속도가 압도적으로 우수**

---

### 3. 종합 인사이트
- **SBR:** 동일 쿼리를 재실행하므로, 소스와 레플리카의 CPU/연산 자원을 이중으로 사용 → **지연 증가**  
- **RBR:** 결과값 기반 복제로 레플리카 부하 감소 → **복제 속도 향상**, 다만 변경 행이 많을 경우 **binlog 크기 증가**

**요약 결론:**  
> 본 실험을 통해 **데이터 무결성과 복제 성능 극대화를 위해 현대적인 환경에서는 RBR이 필수적**임을 확인하였음.


---

