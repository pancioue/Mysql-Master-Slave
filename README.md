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
* STOP SLAVE;
停止整个从库复制进程，包括以下两部分：
IO_THREAD：负责从主库读取二进制日志（binlog）。
SQL_THREAD：负责执行 IO_THREAD 下载到从库的中继日志（relay log）。
效果：同时停止 IO 和 SQL 线程。
用途：通常用于暂停整个从库的同步。
常见场景：
数据迁移或维护。
暂时中断整个同步过程。


* STOP SLAVE SQL_THREAD;
仅停止SQL线程，即停止从库对中继日志的执行。
IO线程会继续运行并从主库拉取新的二进制日志。
用途：用于暂时暂停从库的 SQL 执行，同时让日志同步继续，以便稍后恢复。
常见场景：
需要对从库数据进行分析或操作，避免新日志干扰。
在主库和从库之间数据不一致时，用于暂停执行以进行检查。

* 状态变化
  执行 SHOW SLAVE STATUS 时，你可以查看以下字段状态：
  `Slave_IO_Running` 和 `Slave_SQL_Running`：
   - STOP SLAVE：两者都会显示为 No。
   - STOP SLAVE SQL_THREAD：Slave_SQL_Running 为 No，但 Slave_IO_Running 仍为 Yes。
