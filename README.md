# MySQL Backup Automation

## 📌 프로젝트 개요
본 프로젝트는 MySQL 데이터베이스 백업 자동화 환경을 구축한 사례입니다.  
사원 데이터(부서, 급여, 보너스 등)를 포함한 중요한 업무용 DB를 대상으로, 주기적인 전체 논리 백업을 수행하고 안전하게 보관하는 것을 목표로 합니다.

- **백업 방식**: `mysqldump` 기반 논리 백업 (Full Backup)  
- **보관 구조**: 날짜별 디렉토리 생성 → 해당 날짜의 백업 파일 저장  
- **자동화 도구**: `cron`을 활용하여 매일 새벽 4시에 실행  

---

## 📂 디렉토리 구조

```

mysql-backup/
├── 20250908
│ └── mysql_backup_20250908_165701.tar.gz
├── mysql-backup.sh
└── mysql-backup-log.log

```



- `YYYYMMDD/` : 날짜별 백업 디렉토리  
- `mysql_backup_YYYYMMDD_HHMMSS.tar.gz` : 압축된 SQL 덤프 파일  
- `mysql-backup.sh` : 백업 실행 스크립트  
- `mysql-backup-log.log` : 자동화 로그 파일  

---

## ⚙️ 스크립트 설명 (`mysql-backup.sh`)
```bash
#!/bin/bash

MYSQL_USER="root"      
MYSQL_PASS="root"      
MYSQL_DB="sqld"        
BACKUP_BASE="/home/ubuntu/03.sh/mysql-backup"

DATE=$(date +%Y%m%d_%H%M%S)
DAY=$(date +%Y%m%d)
BACKUP_PATH="$BACKUP_BASE/$DAY"

DB_DUMP="mysql_backup_$DATE.sql"
DB_TAR="mysql_backup_$DATE.tar.gz"

mkdir -p "$BACKUP_PATH"

# 1. DB 덤프
mysqldump -u"$MYSQL_USER" -p"$MYSQL_PASS" "$MYSQL_DB" > "$BACKUP_PATH/$DB_DUMP"

# 2. 덤프 파일 압축
tar -czf "$BACKUP_PATH/$DB_TAR" -C "$BACKUP_PATH" "$DB_DUMP"
rm "$BACKUP_PATH/$DB_DUMP"

echo "[$DAY] MySQL DB 백업 완료: $BACKUP_PATH/$DB_TAR"

```


---


## ⏰ 자동화 설정 (cron)

매일 새벽 4시에 스크립트가 실행되도록 설정했습니다.

```cron
0 4 * * * /home/ubuntu/03.sh/mysql-backup/mysql-backup.sh >> /home/ubuntu/03.sh/mysql-backup/mysql-backup-log.log 2>&1
```

### ✅ 실행 주기: 하루 1회


사원 데이터베이스는 부서, 급여, 보너스 등 **민감한 인사·재무 정보를 포함**하고 있습니다. <br>
이러한 데이터는 기업 운영 및 법적 준거성에 직접적으로 연결되므로, 데이터 손실 허용 범위(RPO, Recovery Point Objective)를 1일로 설정하는 것이 합리적입니다. <br>
즉, 장애 발생 시 복구 가능한 시점이 최대 하루 전까지라는 것을 의미하며, 이를 위해 하루 한 번 전체 논리 백업을 수행합니다. <br>

<img width="1095" height="258" alt="image" src="https://github.com/user-attachments/assets/fc5b154c-7ed2-4605-8b62-07714157724c" />

###### 출처 : https://www.percona.com/blog/mysql-backup-and-recovery-best-practices/



### ✅ 실행 시간: 새벽 4시


백업 작업은 CPU, 메모리, 디스크 I/O에 큰 부하를 줄 수 있으므로 업무 시간대에는 서비스 성능 저하를 유발할 위험이 있습니다. <br>
따라서 **사용자 접속 및 DB 변경 가능성이 가장 낮은 시간대를 선택**해야 합니다. <br>
글로벌 및 국내 인터넷 트래픽 분석 자료에 따르면, 새벽 3~5시 구간은 전체 접속률이 가장 낮은 시간대로 보고되고 있으며, 이 시각은 사원 데이터베이스의 변경 가능성도 사실상 거의 없습니다. <br>
이에 따라 백업을 매일 새벽 4시에 실행하도록 설정했습니다. <br>

<img width="854" height="121" alt="image" src="https://github.com/user-attachments/assets/7ddb727a-e439-4fc6-9edb-e9cf3a7820e1" />

###### 출처 : https://blog.bytescrum.com/how-to-automate-mysql-database-backups-on-any-operating-system

---

## 💾 백업 파일 사용 방법 (복구 절차)

1. 백업 파일 압축 해제

```
tar -xzf mysql_backup_20250908_165701.tar.gz
```

→ mysql_backup_20250908_165701.sql 생성됨

2. 기존 DB로 복원

```
mysql -u root -p sqld < mysql_backup_20250908_165701.sql
```


3. 새로운 DB에 복원 (예: sqld_restore)

```
mysql -u root -p -e "CREATE DATABASE sqld_restore;"
mysql -u root -p sqld_restore < mysql_backup_20250908_165701.sql
```

