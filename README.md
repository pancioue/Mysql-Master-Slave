# Mysql-Master-Slave
1. 在 master 的 server 執行的 SQL command 會被記錄在 Binary Log 裡面
2. master 將 Binary Log 傳送到 slave 的 Relay Log
3. slave 依據 Relay Log 做資料的變更

## 安裝
可以參考  
https://medium.com/dean-lin/%E6%89%8B%E6%8A%8A%E6%89%8B%E5%B8%B6%E4%BD%A0%E5%AF%A6%E4%BD%9C-mysql-master-slave-replication-16d0a0fa1d04

在 Slave 下 `SHOW SLAVE STATUS`  
`Slave_IO_Running` 與 `Slave_SQL_Running` 都為 Yes表示運作正常
