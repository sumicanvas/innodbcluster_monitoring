
# InnoDB Cluster 모니터링 가이드 (MySQL 8.0 / 8.4 기준)

고가용성(High Availability) 환경을 구축 후 운영 시 가장 중요한 것은 **문제가 생기기 전에 미리 감지하고 대응할 수 있는 모니터링 체계**입니다.
MySQL InnoDB Cluster는 강력한 자동 장애 조치(auto failover) 기능을 제공하지만, 네트워크나 하드웨어 문제로도 장애는 발생할 수 있기때문에 **클러스터 상태를 모니터링하는 것은 운영에 있어서 중요한 부분입니다.**

이 글에서는 InnoDB Cluster 모니터링 방법에 대해 설명합니다.

---

## InnoDB Cluster 

<img width="878" height="916" alt="image" src="https://github.com/user-attachments/assets/bf6b2f44-b859-4b8e-92cf-db9b7d418028" />  

InnoDB Cluster는 자동으로 다음을 제공합니다.

* Auto Failover
* 자동 멤버 관리
* 자동 rejoin
* 데이터 충돌 처리

운영 환경에서 발생할 수 있는 이슈

* 특정 노드에서 발생하는 apply 지연
* 네트워크 지연으로 인한 동기화 지연
* Primary가 자주 변경되는 불안정한 환경
* Disk I/O 문제로 relay log 적용이 밀리는 현상

이런 이슈들을 **모니터링을 통해 대처할 수 있습니다.**

---

##  1. MySQL Shell을 활용한 실시간 모니터링

MySQL Shell은 InnoDB Cluster의 관리 및 상태를 모니터링할 수 있는 툴입니다. 뿐만아니라 기존의 mysql client 를 대체할 수 있는 새로운 mysql client 툴로 SQL 모드를 이용하시면 기존 mysql client에서 사용했던 명령어들을 모두 사용하실 수 있습니다.

### ■ 클러스터 전체 상태

```js>
dba.getCluster().status()
또는
dba.getCluster().status({extended:1})
```

<img width="614" height="729" alt="image" src="https://github.com/user-attachments/assets/173fc202-d380-4212-b28d-72a5460dcdaa" />


확인 가능 항목:

| 항목          | 설명                          |
| ----------- | --------------------------- |
| status      | ONLINE / OFFLINE            |
| topology    | Primary / Secondary 역할      |
| mode       |  R/W / R/O            |
| statusText   | GR 오류 메시지 포함                |

**extended 모드**는 delay, transaction queue 등 보다 상세한 정보를 보여줍니다.

---

## 2. replication_group_members – 기본 상태 확인

performance_schema의 replication_group_members 테이블을 통해 클러스터 멤버의 온라인 여부를 확인합니다.

```sql
SELECT MEMBER_ID, MEMBER_HOST, MEMBER_PORT, MEMBER_STATE
FROM performance_schema.replication_group_members;
```
<img width="898" height="199" alt="image" src="https://github.com/user-attachments/assets/cfd2965e-ea3f-44f3-b361-4b8d4f626be4" />


| MEMBER_STATE | 의미         |
| ------------ | ---------- |
| ONLINE       | 정상         |
| RECOVERING   | 복구/재join 중 |
| OFFLINE      | 통신 불가      |
| ERROR        | 비정상 상태     |


---

## 3. replication_group_member_stats – 핵심 지표

InnoDB Cluster의 내부 동작을 확인할 수 있는 가장 중요한 뷰입니다. 특히 아래표의 내용들을 위주로 모니터링하면 좋을 것 같습니다.

```sql
SELECT * FROM performance_schema.replication_group_member_stats\G
```
<img width="832" height="347" alt="image" src="https://github.com/user-attachments/assets/9c398a68-dfc1-44f0-8b58-354761ccf870" />

| 컬럼                                         | 설명                            |
| ------------------------------------------ | ----------------------------- |
| COUNT_TRANSACTIONS_IN_QUEUE                | 적용 대기 중인 트랜잭션 수               |
| COUNT_TRANSACTIONS_CHECKED                 | Certification 단계에서 검사된 트랜잭션 수 |
| COUNT_CONFLICTS_DETECTED                   | 감지된 충돌 횟수                     |
| COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE | 원격 트랜잭션 대기 수                  |
| COUNT_TRANSACTIONS_REMOTE_APPLIED          | 적용된 remote 트랜잭션 수             |
| COUNT_TRANSACTIONS_LOCAL_PROPOSED          | 해당 노드에서 발생한 트랜잭션 수            |
| LAST_CONFLICT_FREE_TRANSACTION             | 마지막으로 충돌 없이 처리된 트랜잭션          |

### 이 뷰를 통해 확인할 수 있는 것들

* 특정 노드에서만 apply delay 커지는 원인
* 네트워크 이슈로 인한 지연


---

## 4. Router 모니터링


### ■ Router 로그 체크 
  
다음 메세지들이 로그에 확인된다면 문제가 발생했을 가능성이 높습니다.

| 메시지                          | 의미                             |
| ---------------------------- | ------------------------------ |
| Metadata cache update failed | Router가 Cluster metadata 접근 실패 |
| All R/W backends unavailable | Primary를 찾지 못함                 |
| Server down                  | 특정 노드 장애 감지                    |


---

# 5. 운영 환경 팁

InnoDB Cluster 는 운영 시 주의가 필요합니다. 아래 주의사항을 꼭 숙지 후 운영하시길 바랍니다.

  
### ✔ 서버를 내리는 순서가 중요함

InnoDB Cluster 전체노드를 shutdown할 때는 **반드시 seconday 노드부터 내려야 하며, 한개 노드가 완전히 종료된 것을 확인 후 다음 노드를 내립니다. 그리고 마지막으로 primary 노드를 종료합니다.**   
```
ps -ef |grep mysqld
```  

### ✔ 정기점검 등의 목적으로 전체 클러스터를 내린 경우 ** 반드시 다음 명령어로 cluster 를 재시작합니다. **
```
dba.dba.rebootClusterFromCompleteOutage()
```

### 클러스터에 한개 노드만 ONLINE 상태인데 mode를 확인했을 때 R/O인 경우, 다음 명령어를 이용해 R/W 모드로 변경

```
var c = dba.getCluster()
c.forceQuorumUsingPartitionOf('사용자ID@해당인스턴스host명:포트')
```





