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
這樣表示只有一 個 process 在處理，可是看 Seconds_Behind_Master 參數是0表示沒有延遲，可能是作者一瞬間塞太多sql

** _目前沒有遇到這樣的狀況，不過若之後有出錯修復，倒是可以留意 Seconds_Behind_Master 的值_ **
