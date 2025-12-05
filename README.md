
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


---

## 4. InnoDB Cluster 메타데이터 


InnoDB Cluster의 메타데이터를 관리하는 테이블은 mysql_innodb_cluster_metadata 스키마 내에 clusters 입니다. 내용이 많아서 다 확인하기는 어렵지만 InnoDB Cluster를 오동작해 메타데이터가 맞지 않을 경우 이 스키마를 삭제하고 InnoDB Cluster를 재구축해야 할 수 있습니다.
클러스터에 대한 전체 메타 데이터가 담겨있는 테이블입니다.

```sql
SELECT * FROM mysql_innodb_cluster_metadata.clusters\G
```
<img width="865" height="649" alt="image" src="https://github.com/user-attachments/assets/c38237a1-adb0-44a5-82a0-31b6b00f0753" />  

### 클러스터 기본 정보
| 옵션명            | 설명                                                    | 
| -------------- | ----------------------------------------------------- | 
| `cluster_id`   | 클러스터를 식별하는 UUID                                       | 
| `cluster_name` | 클러스터 이름(MySQL Shell에서 표시됨)                            | 
| `cluster_type` | 클러스터 유형 (`gr` = Group Replication)                    | 
| `primary_mode` | Primary 모드 (`pm`= single-primary, `mp`=multi-primary) | 
| `description`  | 클러스터 설명(옵션)                                           | 
| `options`      | 예약 옵션 (일반적으로 NULL)                                    | 


### attributes 필드 주요 옵션 + 기본값  
  
| 옵션명                                       | 설명                                       | 기본값(Default)          |
| ----------------------------------------- | ---------------------------------------- | --------------------- |
| `adopted`                                 | 기존 GR 클러스터를 Shell에서 가져왔는지 여부             | `0`                   |
| `default`                                 | 기본(Default) 클러스터인지 여부                    | `true`                |
| `opt_memberAuthType`                      | GR 멤버 간 인증 타입 (`PASSWORD`, `X509`)       | `PASSWORD`            |
| `opt_gtidSetIsComplete`                   | GTID 세트가 완전한지 여부 (Auto failover 안정성에 영향) | `false`               |
| `opt_manualStartOnBoot`                   | MySQL 서버 재시작 시 GR 자동 시작 여부               | `false`               |
| `opt_transactionSizeLimit`                | 단일 트랜잭션 최대 크기(Byte 단위)                   | `150000000` (약 150MB) |
| `opt_replicationAllowedHost`              | GR 멤버 조인 허용 호스트 (IP 또는 `%`)              | `%`                   |
| `capabilities → communicationStack.value` | GR 통신 스택                                 | `MYSQL`               |
| `group_replication_group_name`            | GR 그룹 UUID                               | 자동 생성 값               |

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





