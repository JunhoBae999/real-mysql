# DB 백업가이드
특정 DBMS가 아닌 일반적인 RDBMS를 대상으로 하는 DB에 대한 전반적인 백업 가이드

[image:DFC4E7D3-A3EB-42ED-983F-E0DACFF3561A-24848-00000F4EB4F3C793/6D9FA56E-FC86-41AE-B56F-6D51156DD434.png]

> 참조 : https://dataonair.or.kr/db-tech-reference/d-lounge/technical-data/?mod=document&uid=235704

> 개발자가 언제 죽는다고 생각하나? 총알이 심장을 관통했을 때? 아니야..
불치병에 걸렸을 때? 아니지! 맹독 버섯스프를 마셨을때? 아니다!!
그건 바로 메인 DB를 날렸을 때다.

## 장애유형
### 물리적 장애
* 시스템 장애 : 하드웨어 오작동 (DB서버 다운 등)
* 미디어 장애 : 저장장치 고장 (DB가 깨졌다)

### 논리적 장애
* 트랜잭션 장애 : 트랜잭션의 논리적 오류 및 데이터 오입력 등 (인적 장애)

### DB 백업 종류
#### 방법 기준
1. 논리 vs 물리
* 논리 : DB서버에서 담당(환경파일, 로그파일 등 제외)
* 물리 : OS에서 통째로

2. 온라인 vs 오프라인
* 온라인 : DB서버 수행중 백업 (핫백업) cf) 웜백업 : 백업 동안 데이터 변경 불가
* 오프라인 : 콜드백업. DB 사용불가 - 실 운영환경에서는 거의 적용 불가

3. 로컬 vs 원격
* 로컬 : 백업 수행하는 공간과 백업본 보관 공간 동일 (DB 서버 호스트에서)
* 원격 : 백업본 별도 보관 (백업 장비, 원격지 클라이언트 등) - 네트워크 환경 민감

####  내용 기준
 1. *Full Backup (전체 백업)*
* 전체 Database 백업. 다른 곳에 복원 시 그대로 사용 가능
* 데이터가 늘어날 수록 백업 시간 증가

2. *Incremental Backup (증분 백업)*
* 이전 백업 시점 이후부터 현재까지 변경된 내역만 백업(직전 백업의 lsn보다 큰 번호만 백업)
* 풀백업보다 백업 시간 짧고, 용량 작음
* DBMS에 따라 지원 여부 다름 (MySQL, Oracle 등)

3. *Differential Backup (차등 백업)*
* 마지막 풀백업 이후부터 변경된 내역만 백업
* 증분 백업보다 복원 시간 빠름
* DBMS에 따라 지원 여부 다름 (Oracle, SQL Server 등)

4. *Log Backup*
* 로그 아카이브 등 DBMS에 따라 구현 방식 다름

### DB 복원 종류

1. 전체 복원
* 풀백업, 증분백업 등을 이용한 복원
* 백업 시점까지만 복원 가능

2. 시점 복원 (PITR : Point In Time Recovery)
* 백업 시점과 상관 없이 특정 시점으로의 복원 가능 ← 로그 백업 전제
로그 기반으로 특정 시간이나 특정 로그번호 지정하여 해당 시점의 로그까지 적용하는 기법

### MySQL 백업 방법
1. Data 디렉토리 백업
* 구분 : 물리, 오프라인
* Data 디렉토리를 정기적으로 백업하고 문제 발생 시 덮어씀
* 속도는 가장 빠르나, 오프라인 필요하고 데이터 이전 어려울 수 있음 (동일 OS 환경 등)

2. mysqldump
* 구분 : 논리, 온라인
* MySQL 기본 백업 툴. 전체 또는 특정 데이터베이스, 특정 테이블만 백업 가능
스키마만 추출 등 여러 옵션 제공
* 데이터 증가할수록 백업/복구 속도 느림
* —single-transaction 및 —flush-logs 옵션 이용하여 백업정책 및 PITR 적용 가능
* 백업 예제 : mysqldump -u root -p --all-databases > mysqldump.sql (이름.Sql)
* 복구 방법 : mysql -u root -p {DB명} < mysqldump.sql

3. xtrabackup
* 구분 : 논리, 온라인
* Percona에서 만든 무료 백업 솔루션
* 데이터 및 로그파일 백업 및 적용을 통해 consistent한 데이터 제공 가능
* 증분백업 및 압축, 암호화 백업, 병렬 처리 등 다양한 기능 제공
* 데이터 증가할수록 mysqldump와 성능 차이 큼
* 백업 예제
	* 풀백업 : innobackupex --user root --password ppp /Backup
	* 증분백업 : innobackupex --user root --password ppp --incremental --incremental-basedir=/이전백업의절대경로/ /Backup
* 복구 방법
	* 풀백업 로그 반영 : innobackupex --apply-log --redo-only /Backup/풀백업디렉토리
	* 증분백업 로그 반영 : innobackupex --apply-log /Backup/풀백업디렉토리 --incremental-dir=/Backup/증분백업디렉토리
	* 복구 : innobackupex --copy-back /Backup/풀백업디렉토리
사례 : https://techblog.woowahan.com/2576/

### 백업 정책 가이드
#### 기본 가이드
* 백업 주기 및 보존기간 정책 수립 및 자동화 설정
* 백업 수행 시간 설정 : DB 사용률 및 성능(디스크 I/O, 네트워크 부하) 고려
* 알림 설정 권장 : 성공 or 실패
* 백업파일은 가급적 원격지 저장 

* 업무수행에 지장을 받지 않는 시간대에 HOT BACKUP을 수행한다.
* 업무변화가 대량으로 발생하기 전에 APPLICATION을 통한 BACKUP수행
다. 
* 자주 read-write되는 tablespace는 자주 online backup을 수행.
*  데이터베이스에 구조적인 변화가 생기기 前,後로 full backup을 수행.
* 이전의 backup본을 최소한 2본 이상 가지고 있을 필요가 있다.
* 특정 테이블들에 대한 data의 입력 오류로 인해 과거 특정 시점으로의 회귀가 필요하거나, 특정 테이블 데이터의 분실로 인해 다시 복귀를 하고자 할 경우를 대비하여 Logical Backup인 Export를 수시로 받아놓도록 한다.
* Unrecoverable로 Creation된 Object는 redo log file에 logging되지 않기에 이러한 Object들에 대해서는 Export Utility를 사용하여 Backup하도록 하는 것이 좋으며, 초기 생성 후 정상적인 데이터 입력/수정이 이루어질 경우에는 logging으로 변경하도록 한다

#### MySQL Binary Log 설정
* 로그 백업 및 PITR 위해서는 Binary Log 설정 필요 (운영환경 등)
* 예제 (my.cnf 설정 - 5.7 버전 기준)
```mysql
[mysqld]
log-bin=/var/log/mysql/bin.log
binlog_cache_size=2M
max_binlog_size=512M
expire_logs_days=7
```

#### 개발, 테스트 환경
* 전체 백업 주기적 실행
* 데이터 사이즈, 중요도에 따라 증분백업, 로그백업 추가 고려

#### 운영 환경
* 전체 백업 + 증분백업 + 로그백업 활용
* 데이터 사이즈 및 성능, 자원 상황에 따라 적절한 백업 정책 수립
* PITR 가능해야 함
* 참조 : https://kakaoenterprise.agit.in/g/300045148/wall/338343851

### 장애유형 별 복구절차
#### 물리적 장애
* DBMS 자체의 복구 기능 이용 (MySQL : innodb_force_recovery)
* 실패 시 DB 백업본 복구 (최종 백업 시점까지 복구 가능)
* 만약 로그파일만이라도 살릴 수 있다면 별도 백업 후에 최종 백업본 복구 후 로그 적용

#### 논리적 장애
* 장애 발생 시점 확인 (매우 중요)
* 장애 발생 시점까지 풀백업 + 증분백업 + 로그백업 복원 수행
* 로그백업이 없는 경우 장애 전 최근 백업본까지만 복구 가능
* 특정 테이블만 복원 시 : 별도 DB 또는 별도 환경에 PITR 수행 후 데이터 복사