---
aliases:
  - msa
  - saga
  - saga-pattern
Date: 2026-01-05
tags:
  - topic/cloud
  - type/summary
---
# Saga Pattern
- 참고 자료: [Microservice Architecture Pattern: Saga](https://microservices.io/patterns/data/saga.html)
## 1. Context & Problem
- Microservice Architecture에서 각 마이크로서비스는 자신만의 데이터베이스를 소유하며, 다른 서비스의 DB에 직접 접근할 수 없다. (Encapsulation)
- 이때 분산 시스템의 일관성을 보장해야 하는데, 두 가지 문제가 발생한다.
	- ACID의 범위 제한
		- 단일 DB와 달리, 서비스 경계를 넘나드는 트랜잭션에서는 로컬 ACID를 적용할 수 없다.
	- 2PC (2 Phase Commit)
		- 분산 트랜잭션의 전통적 해법이나 다음과 같은 이유로 MSA에 적합하지 않다.
			- 모든 노드가 `Prepare`와 `Commit` 단계에서 Globol Lock을 잡고 응답을 대기해야 하므로  성능 병목 발생
			- SPOF (Single Point of Failure)
				- 시스템 내에서 특정 컴포넌트 하나가 고장났을 때, 전체 시스템이 멈춤
			- NoSQL 미지원
## 2. Solution - SAGA
<img width="779" height="465" alt="Pasted image 20260105161107" src="https://github.com/user-attachments/assets/edfe76f7-46da-48f2-8e58-fa1d130d6891" />

- SAGA 패턴은 MSA 환경에서 분산 트랜잭션(Distributed Transaction)의 데이터 일관성(Data Consistency)을 보장하기 위해 설계된 아키텍처 패턴이다.
- Distributed Transaction을 **Sequence** of Local Transactions로 재정의한다.
	1. Each Local Transaction
		- updates the DB
		- publishes a message or event **to trigger the next local transaction** 
	2. Compensating Transaction (보상 트랜잭션)
		- If a local transaction **fails**
		- executes **a series of compensating transactions** that **undo the changes**
			- that were made by the preceding local transactions.
		- 앞서 commit된 변경 사항을 취소하기 위해 역순으로 보상 트랜잭션 실행
## 3.Coordination Sagas: Choreography vs. Orchestration

|       | Choreography                                                                                       | Orchestration                                                                         |
| ----- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
|       | Event Driven                                                                                       | Command Driven                                                                        |
| 제어 주체 | 중앙 제어 X                                                                                            | 중앙 Saga Orchestrator                                                                  |
| Pros  | 서비스간 결합도 낮음 (Loose Coupling)<br>단일 실패 지점(SPOF) 없음                                                  | 흐름이 명확<br>모니터링/관리 용이                                                                  |
| Cons  | Cyclic Dependency 위험                                                                               | 추가적인 복잡성이 생길 수 있음                                                                     |
|       | `Each local transaction publishes domain events that trigger local transactions in other services` | `an orchestrator (object) tells the particiipants what local transactions to execute` |
#### Example: Choreography-based saga
- 중앙 제어 없이, 서비스 (event handler) 간 이벤트를 교환
<img width="1018" height="432" alt="Pasted image 20260106180609" src="https://github.com/user-attachments/assets/c1b450da-fb1e-417c-93b6-814fa532e5f3" />

1. `Order Service`에 `POST /orders` 요청이 들어오면,  `Order` 생성한다. (`PENDING` state)
2. publish an `Order Created` event (trigger)
3. `Customer Service`(subscribe)의 event handler가 `Credit Reserved`를 시도한다.
4. 처리 결과에 따라: publish `Credit Reserved` or `Credit Limit Exceeded`
5. `Order Service`의 event handler가 `order`의 상태 변경
#### Example: Orchestration-based saga
- 중앙의 Saga Orchestrator가 프로세스 조율
<img width="1465" height="608" alt="Pasted image 20260106204430" src="https://github.com/user-attachments/assets/7db97d56-d296-4969-87f5-56d6cbbe2885" />

1. `Order Service`가 `POST /orders` 요청을 받으면 `Create Order Saga` 즉, Orchestrator 생성
2. Orchestrator가 `Order` 생성 (`PENDING`)
3. Orchestrator가 `Customer Service`에 command 전송 (`Reserve Credit Command`)
4. `Customer Service`에서 수신한 후 Orchestrator에 결과 반환
5. Orchestrator가 결과에 따라 `Order`의 상태 변경
### 4. Trade-off
#### 1. Lack of Automatic Rollback
- Local transacion이 실패하면 이미 commit된 변경 사항을 취소하기 위해 compensating transactions를 실행해야 한다. (역순으로)
- 이때 compensating transaction은 개발자가 직접 설계해야 함.
	- *rather than relying on the automatic rollback feature of ACID transactions*
#### 2. Lack of Isolation (the "I" in ACID)
- concurrent execution of multiple sagas and transactions
	- 전체 saga가 완료되지 않은 중간 상태가 다른 transaction에 노출될 수 있다.
		- readable/changable
	- Data Corruption으로 이어질 수 있음.
- Solution: 
	- saga가 완료될 때까지 해당 데이터에 대한 다른 작업의 접근을 논리적으로 차단
	- 연산 순서가 바뀌어도 결과가 같도록 설계
#### 3. Atomically update DB & publish event
- DB 업데이트와 이벤트 publish가 원자적으로 이루어지는데 하나만 실패할 경우 (Inconsistency)
- 데이터 불일치 발생 가능성 $\rightarrow$ 데이터 일관성 깨짐
- Solution:
	- 데이터와 event/message를 하나의 transaction으로 묶어 commit한다.
