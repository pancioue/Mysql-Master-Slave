# Mysql-Master-Slave
1. 在 master 的 server 執行的 SQL command 會被記錄在 Binary Log 裡面
2. master 將 Binary Log 傳送到 slave 的 Relay Log
3. slave 依據 Relay Log 做資料的變更

## 安裝
可以參考  
https://medium.com/dean-lin/%E6%89%8B%E6%8A%8A%E6%89%8B%E5%B8%B6%E4%BD%A0%E5%AF%A6%E4%BD%9C-mysql-master-slave-replication-16d0a0fa1d04

在 Slave 下 `SHOW SLAVE STATUS`，`Slave_IO_Running` 與 `Slave_SQL_Running` 都為 Yes表示運作正常

如何查看 Slave 延遲
--------
一樣是下 `SHOW SLAVE STATUS` 或者在phpmyadmin查看 `Seconds_Behind_Master`。  
正常情況下一般是0

主從複製的延遲問題
--------
https://medium.com/dean-lin/%E5%9C%A8-mysql-5-7-%E8%A7%A3%E6%B1%BA%E4%BA%86%E4%B8%BB%E5%BE%9E%E8%A4%87%E8%A3%BD%E7%9A%84%E5%BB%B6%E9%81%B2%E5%95%8F%E9%A1%8C-67c2aec6b021  
按照這篇所講的，應該會有嚴重的延遲，
`show variables like 'slave_parallel%';`
```
slave_parallel_type    = DATABASE
slave_parallel_workers = 0
```
這樣表示只有一 個 process 在處理，可是看 Seconds_Behind_Master 參數是0表示沒有延遲  
可能是作者一瞬間塞太多sql

** _目前沒有遇到這樣的狀況，不過若之後有出錯修復，倒是可以留意 Seconds_Behind_Master 的值_ **

錯誤
--------
在執行 `SHOW SLAVE STATUS` 時，會看到 `Slave_SQL_Running: No`，並且 `Last_SQL_Error` 中會顯示錯誤訊息  
phpmyadmin 狀態會顯示 Last_Error，但可能會比較攏統(也可能一樣)，具體原因可能要看 Last_SQL_Error
* 如果 Last_Error 和 Last_SQL_Error 都有值，應優先檢查 Last_SQL_Error，因為它通常更具體。

排除
--------
* 如果錯誤可以忽略
  ```
  STOP SLAVE SQL_THREAD;
  SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
  START SLAVE SQL_THREAD;
  ```
  ** _無法直接查看 SQL_SLAVE_SKIP_COUNTER，SQL_SLAVE_SKIP_COUNTER 不是持久參數，僅在執行跳過操作時生效，無法查詢其當前值_ **
* 如果知道明確的錯誤原因
  ```
  STOP SLAVE SQL_THREAD;
  INSERT INTO `test` VALUE ('16', 'apple', 350); // 解決實際問題的操作
  START SLAVE SQL_THREAD;
  ```
可以參考: https://www.cnblogs.com/xuanzhi201111/p/4566451.html  
這網站列舉了一些壞掉的例子

STOP SLAVE VS STOP SLAVE SQL_THREAD
--------
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
  - 需要對從庫資料進行分析或操作，避免新日誌幹擾。
  - 在主庫和從庫之間資料不一致時，用於暫停執行以進行檢查。

* 狀態變化
  執行 SHOW SLAVE STATUS 時，你可以查看以下欄位狀態：
  `Slave_IO_Running` 和 `Slave_SQL_Running`：
   - STOP SLAVE：兩者都會顯示為 No
   - STOP SLAVE SQL_THREAD：Slave_SQL_Running 為 No，但 Slave_IO_Running 仍為 Yes
