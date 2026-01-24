> Mysql版本: 5.7  
> MySQL 主從複寫中，主庫將資料變更以事件寫入 Binary Log，從庫透過 I/O thread 讀取並寫入 Relay Log，再由 SQL thread 執行以同步資料。

# 主從設定
Step 1️⃣ 設定 MySQL 參數（需重啟）
主庫設定
```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
```

從庫最小設定
```ini
[mysqld]
server-id = 2
relay-log = mysql-relay-bin
read_only = ON
```

- - -
Step 2️⃣ 驗證主庫 binlog 已啟用（不中斷）
```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW MASTER STATUS;
```
確認：
* log_bin = ON
* 有 binlog file / position

- - -
Step 3️⃣ 在主庫建立 replication 使用者（不中斷） 
```
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```
- - -
Step 4️⃣ 建立一致性資料快照（不中斷）
```
mysqldump \
  --single-transaction \
  --master-data=2 \
  --routines --triggers \
  --databases your_db > dump.sql
```
✔ 不鎖 InnoDB
✔ 不影響線上寫入

* --single-transaction: 不鎖 InnoDB 表
  在 dump 開始時：
    - 開一個 transaction
    - 使用 REPEATABLE READ
  整個 dump 期間：
    - 看到的是「同一時間點的資料」
* --master-data=2: 
  - 記錄 「這份 dump 對應主庫的哪個 binlog 位置」
  - 這是從庫能安全接上 replication 的 唯一依據
* --routines:
  匯出：
    - Stored Procedure
    - Function
* --triggers: 匯出 table trigger

- - -
Step 5️⃣ 從庫還原資料
```bash
mysql -h localhost -u root -p < dump.sql
```

- - -
Step 6️⃣ 設定從庫 replication 起點
```
CHANGE MASTER TO
  MASTER_HOST='master_ip',
  MASTER_USER='repl',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000123',
  MASTER_LOG_POS=456789;
```

- - -
Step 7️⃣ 啟動 replication
```sql
START SLAVE;
SHOW SLAVE STATUS\G
```

確認：
* Slave_IO_Running: Yes
* Slave_SQL_Running: Yes

# 狀況：緊急停止主庫，全部轉移到從庫
```
STOP SLAVE;
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;   -- 如果有開
```
這樣可以不用停止從庫運行，不過之後要再設定主從就要記得把設定檔改掉


# 除錯
> 在執行 `SHOW SLAVE STATUS` 時，會看到 `Slave_SQL_Running: No`，並且 `Last_SQL_Error` 中會顯示錯誤訊息  
  phpmyadmin 狀態會顯示 Last_Error，但可能會比較攏統(也可能一樣)，具體原因可能要看 Last_SQL_Error

* 如果 Last_Error 和 Last_SQL_Error 都有值，應優先檢查 Last_SQL_Error，因為它通常更具體。

👉 如果錯誤可以忽略
```
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START SLAVE;
```
* 無法直接查看 SQL_SLAVE_SKIP_COUNTER，SQL_SLAVE_SKIP_COUNTER 不是持久參數，僅在執行跳過操作時生效，無法查詢其當前值

👉 如果知道明確的錯誤原因
```
INSERT INTO `test` VALUE ('16', 'apple', 350); // 解決實際問題的操作
START SLAVE;
```
可以參考: https://www.cnblogs.com/xuanzhi201111/p/4566451.html  
這網站列舉了一些壞掉的例子

- - -
從庫技術上可以指定到某個 binlog 座標繼續開始，  
但實務上很少「憑空指定」座標；一定要有「可信的對齊依據」
```
CHANGE MASTER TO
  MASTER_LOG_FILE = 'mysql-bin.000001',
  MASTER_LOG_POS = 154;
```

如果真的亂掉救不回來，只能重灌了
* 利用 mysqldump 重灌數據
* 重新設置從庫座標

- - -

STOP SLAVE VS STOP SLAVE SQL_THREAD
* STOP SLAVE
  停止整個從庫複製進程，包括以下兩部分：
  IO_THREAD：負責從主庫讀取二進位日誌（binlog）。
  SQL_THREAD：負責執行 IO_THREAD 下載到從庫的中繼日誌（relay log）。
  效果：同時停止 IO 和 SQL 執行緒。
  用途：通常用於暫停整個從庫的同步。  
  常見場景：
  - 資料遷移或維護。
  - 暫時中斷整個同步過程。

* STOP SLAVE SQL_THREAD
  僅停止SQL線程，即停止從庫對中繼日誌的執行。
  IO線程會繼續運行並從主庫拉取新的二進位日誌。
  用途：用於暫時暫停從庫的 SQL 執行，同時讓日誌同步繼續，以便稍後恢復。  
  常見場景：
  - 需要對從庫資料進行分析或操作，避免新日誌干擾。
  - 在主庫和從庫之間資料不一致時，用於暫停執行以進行檢查。

* 狀態變化
  執行 SHOW SLAVE STATUS 時，你可以查看以下欄位狀態：
  `Slave_IO_Running` 和 `Slave_SQL_Running`：
   - STOP SLAVE：兩者都會顯示為 No
   - STOP SLAVE SQL_THREAD：Slave_SQL_Running 為 No，但 Slave_IO_Running 仍為 Yes

- - -
event 是什麼？
> binlog 裡的一個記錄單位

它可能是：
* 一條 SQL
* 一筆 row change
* 一個 table mapping
* 一個 commit

目前主流是 GTID replication
