# 📊 MySQL 복제 포맷(SBR vs RBR) 성능 비교

## 테스트 개요
MySQL의 `binlog_format` 설정에 따른 복제 메커니즘의 차이를 이해하고,  
실제 운영 환경에서 발생할 수 있는 **데이터 불일치** 및 **복제 지연(Replication Lag)** 현상을  
실험을 통해 분석합니다.

## 3️⃣ 실행 시간 비교

### 1. 테스트 환경 및 조건
- **대상 테이블:** `time_test` (약 20,000건의 레코드)  
- **부하 쿼리:**  
  ```sql
  UPDATE ... SET ... WHERE SLEEP(0.0005) = 0;
  ```
- **소스 서버 실행 시간:** 약 **17.68초**
    <img width="651" height="141" alt="image" src="https://github.com/user-attachments/assets/10cdb8f9-b42e-49d6-9f0b-d9a423028eaf" />
    
- **측정 지표:** `SHOW REPLICA STATUS` 내 `Seconds_Behind_Source`
    <img width="795" height="103" alt="image" src="https://github.com/user-attachments/assets/8c07d42e-51c3-4a14-b022-4e5ef1494dc7" />

---

### 2. 포맷별 지연 시간(Latency) 비교

#### 🔹 [Case 1] SBR (Statement-Based Replication)
- **지연 시간 추이:** `19s → 37s → 0s`<br>
    ![result](https://github.com/user-attachments/assets/754c2c49-3f30-4a06-b664-be3f5b9df419)


**현상 분석**
- 소스 서버에서 쿼리가 완료되어 로그가 전달된 시점에 이미 약 **19초의 지연** 발생  
- 레플리카가 전달받은 SQL을 실행하는 동안 실제 시간이 계속 흐르며 **지연 수치가 37초까지 증가**  
- 레플리카가 소스와 동일한 연산(`SLEEP`)을 모두 마친 후에야 지연이 `0s`로 수렴  

**결론:**  
> 소스의 연산 부하가 레플리카에 그대로 전이되어 **복제 지연(Lag)** 이 심화됨

---

#### 🔹 [Case 2] RBR (Row-Based Replication)
- **지연 시간 추이:** `19s → 1~2s → 0s (혹은 즉시 0s)`<br>
    <img width="326" height="34" alt="image" src="https://github.com/user-attachments/assets/fce3e55d-b977-437f-8a71-1337a174271b" />


**현상 분석**
- 소스에서 쿼리가 끝난 시점에 발생한 **기본 지연(약 19초)** 은 동일  
- 하지만 레플리카는 연산 과정을 생략하고 **최종 변경된 값만 즉시 반영**  
- 쿼리 수신 직후 지연 시간이 급격히 감소  

**결론:**  
> 연산량이 많은 작업에서는 **RBR이 SBR보다 복제 성능과 동기화 속도가 압도적으로 우수**

---
## 🧪 실험 요약 (Test Summary)

본 실험은 **MySQL 복제 아키텍처**에서 `SBR`과 `RBR` 설정이 시스템 운영에 미치는 영향을  
**[로그 용량 / 데이터 정합성 / 복제 성능]** 세 가지 관점에서 비교·분석하였습니다.

| 실험 항목 | 측정 지표 | SBR 결과 | RBR 결과 | 비고 |
|------------|------------|------------|------------|------------|
| **1. 데이터 정확성** | `UUID()` 함수 결과 비교 | 불일치 (초 단위 차이) | 완벽 일치 | SBR 사용 시 데이터 오염 위험 |
| **2. 로그 파일 사이즈** | 파일 용량 | 매우 작음 (수 KB) | 큼 (수 MB) | 대량 데이터 변경 시 RBR 부하 증가 |
| **3. 실행 시간(지연)** | `Seconds_Behind_Source` | 37초까지 누적 지연 | 즉각 반영 (0~1초) | 연산량이 클수록 RBR 압도적 우세 |

---
## 💡 종합 인사이트 (Final Conclusion)

실험 결과, 각 복제 방식은 명확한 **Trade-off**를 가지고 있습니다.


### 🔹 SBR (Statement-Based Replication)
- **장점:** 로그 용량이 작아 저장 공간 절약에 유리  
- **단점:**  
  - 데이터 정합성이 깨질 위험이 큼  
  - 복잡한 연산 시 **심각한 복제 지연(Replication Lag)** 발생  


### 🔹 RBR (Row-Based Replication)
- **장점:**  
  - 어떤 상황에서도 **데이터 무결성(Data Integrity)** 보장  
  - 연산 부하가 큰 환경에서도 **실시간에 가까운 복제 성능** 유지  
- **단점:** 변경된 행 단위로 로그를 기록하므로 **binlog 파일 크기 증가**

---

### ✅ 결론
> 현대의 엔터프라이즈 환경에서는 **데이터 정확성과 시스템의 고가용성(High Availability)** 이 최우선입니다.  
> 따라서 **로그 용량의 증가를 감수하더라도 `RBR (Row-Based Replication)` 방식을 채택**하는 것이 권장됩니다.
