
# InnoDB Cluster 모니터링 가이드 (MySQL 8.0 / 8.4 기준)

고가용성(High Availability) 환경을 구축할 때 가장 중요한 것은 **문제가 생기기 전에 미리 감지하고 대응할 수 있는 모니터링**입니다.
MySQL InnoDB Cluster(Group Replication 기반)는 강력한 자동 장애 조치(auto failover) 기능을 제공하지만,
**클러스터 상태를 모니터링하는 것은 서비스에 있어서 중요한 부분입니다.**

이 글에서는 InnoDB Cluster 운영자가 반드시 체크해야 할 **모니터링 포인트, 주요 Performance Schema 뷰, 유용한 Shell 명령어**를 설명합니다.

---

## InnoDB Cluster 
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

#  1. MySQL Shell을 활용한 실시간 모니터링

MySQL Shell은 InnoDB Cluster 상태를 가장 보기 좋게 보여주는 도구입니다.

### ■ 클러스터 전체 상태

```js>
dba.getCluster().status()
또는
dba.getCluster().status({extended:1})
```

확인 가능 항목:

| 항목          | 설명                          |
| ----------- | --------------------------- |
| status      | ONLINE / OFFLINE            |
| topology    | Primary / Secondary 역할      |
| memberState | ONLINE / ERROR / RECOVERING |
| lag         | Apply 지연 시간                 |
| lastError   | 최근 GR 오류 메시지                |

**extended 모드**는 delay, transaction queue 등 보다 상세한 정보를 보여줍니다.

---

# 2. replication_group_members – 기본 상태 확인

performance_schema의 replication_group_members 테이블을 통해 클러스터 멤버의 온라인 여부를 확인합니다.

```sql
SELECT MEMBER_ID, MEMBER_HOST, MEMBER_PORT, MEMBER_STATE
FROM performance_schema.replication_group_members;
```

| MEMBER_STATE | 의미         |
| ------------ | ---------- |
| ONLINE       | 정상         |
| RECOVERING   | 복구/재join 중 |
| OFFLINE      | 통신 불가      |
| ERROR        | 비정상 상태     |


---

# 3. replication_group_member_stats – 핵심 지표 모음

InnoDB Cluster의 내부 동작(certificate, apply queue)을 확인할 수 있는 가장 중요한 뷰입니다.

```sql
SELECT * FROM performance_schema.replication_group_member_stats\G
```

| 컬럼                                         | 설명                            |
| ------------------------------------------ | ----------------------------- |
| COUNT_TRANSACTIONS_IN_QUEUE                | 적용 대기 중인 트랜잭션 수               |
| COUNT_TRANSACTIONS_CHECKED                 | Certification 단계에서 검사된 트랜잭션 수 |
| COUNT_CONFLICTS_DETECTED                   | 감지된 충돌 횟수                     |
| COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE | 원격 트랜잭션 대기 수                  |
| COUNT_TRANSACTIONS_REMOTE_APPLIED          | 적용된 remote 트랜잭션 수             |
| COUNT_TRANSACTIONS_LOCAL_PROPOSED          | 해당 노드에서 발생한 트랜잭션 수            |
| LAST_CONFLICT_FREE_TRANSACTION             | 마지막으로 충돌 없이 처리된 트랜잭션          |

### ✔️ 이 뷰로 진단할 수 있는 것들

* 특정 노드에서만 apply delay 커지는 원인
* certification 충돌 증가 여부
* 네트워크 이슈로 인한 지연
* Primary failover 이후 특정 멤버만 재join 느린 이유

InnoDB Cluster 운영자에게 가장 중요한 성능 뷰입니다.

---

# 🧩 4. Router 모니터링

MySQL Router는 단순 Proxy처럼 보이지만, 실제로는 클러스터 상태 기반 라우팅을 처리하는 중요한 레이어입니다.

### ■ Router 상태 점검

```bash
mysqlrouter -s
```

확인 가능 항목:

* metadata 연결 여부
* R/W backend 가용 여부
* Router bootstrap 정보
* routing endpoint 상태

### ■ Router 로그 체크 (중요)

다음 로그가 보이면 주의가 필요합니다:

| 메시지                          | 의미                             |
| ---------------------------- | ------------------------------ |
| Metadata cache update failed | Router가 Cluster metadata 접근 실패 |
| All R/W backends unavailable | Primary를 찾지 못함                 |
| Server down                  | 특정 노드 장애 감지                    |

운영 환경에서는 **Router 로그 모니터링**도 필수입니다.

---

# 🧩 5. 모니터링 포인트 요약

아래는 실무에서 자주 쓰는 핵심 체크 포인트입니다.

| 모니터링 항목           | 뷰/명령어                          | 알람 기준                  |
| ----------------- | ------------------------------ | ---------------------- |
| 멤버 온라인 여부         | replication_group_members      | OFFLINE 시 즉시 알람        |
| apply delay       | replication_group_member_stats | delay > 1~2초           |
| certification 충돌  | COUNT_CONFLICTS_DETECTED       | 0 이상 지속 증가             |
| Router backend 상태 | mysqlrouter -s                 | RW backend UNAVAILABLE |
| Primary 변경        | View_ID 변화                     | 자주 바뀌면 네트워크 문제         |
| 시스템 OS 상태         | CPU/IO 지표                      | apply delay와 직결        |

---

# 🧩 6. 운영 환경 팁

### ✔ Primary 변경이 자주 발생하면 네트워크 품질을 의심

`VIEW_ID` 변화가 자주 일어나면 노드 간 통신 품질 문제일 가능성이 높습니다.

### ✔ Router는 가급적 2~3대 이상

MySQL Router는 stateless하므로 여러 대 띄워두면 장애에 매우 강합니다.

### ✔ apply queue 증가 → 디스크 I/O 의심

특정 노드만 지연이 심하면 대부분 디스크 성능 이슈입니다.

---

# 🏁 마무리

InnoDB Cluster는 자동화된 고가용성 기능을 갖추고 있지만,
**클러스터 내부에서 어떤 일이 일어나고 있는지 모니터링하는 것이 운영 안정성의 핵심**입니다.

이번 글에서 소개한 Performance Schema 뷰와 Shell 명령어만 잘 활용해도
장애의 대부분을 빠르게 감지하고 대응할 수 있습니다.

다음 포스팅에서는 실 운영에서 자주 발생하는 InnoDB Cluster 장애 사례와
**트러블슈팅 가이드**를 소개하겠습니다.

