# learning-Temporal

Dashboard Web: http://localhost:8088/

### Question

1. 要如何處裡每一個 workflow 的版本差異?
3. 每一個 workflow history 的最大可以保存的資料量是多少?
4. 如何沒有影響的修改 cron workflow 的 schedule?
5. 需要研究如何傳遞 `request_id` 到每個 workflow 裡面

### Client

1. 可以先把任務發給 temporal server, `worker` 並不需要起來，之後可以等 `worker` 起來在執行

### Worker

1. 每一個 worker 預設是 `2` 個 goroutines 從 server 拉任務 (MaxConcurrentWorkflowTaskPollers)
2. 每一個 worker 預設同一時間最多執行 `1000` activity   (MaxConcurrentActivityExecutionSize)

### Workflow

1. Workflow 是由 `workflow worker` 來執行的
2. Workflow ID 是唯一的。Temporal guarantees that there could be only one workflow (across all workflow types) with a given Id open per [namespace](https://docs.temporal.io/docs/learn-glossary#namespace) at any time. An attempt to start a workflow with the same Id is going to fail with `WorkflowExecutionAlreadyStarted` error.
3. 如果中間 `workflow worker` 掛掉了，整個 workflow 會重跑，但 `activity` 會從 history 找之前的結果
4. 過程中如果有用到 logger, 當重複執行他不會列印出來內容 (好像可以設定要 reply)
5. Go SDK, goroutine 和 sleep 等一些 func 需要改用 workflow SDK 裡面的對應func
   https://docs.temporal.io/docs/go-create-workflows/#special-temporal-sdk-functions-and-types
   舉例: 假設一個場景，我們想讓某個 activity 暫時休息 3 秒, 如果使用 `time.Sleep`, 這樣其實 server 是不知道要 sleep, 所以導致 activity 還是會觸發actitity 的timeout 機制, 預設 10 秒, 正確的動作應該要用 `workflow.Sleep`
6. 如果 workflow 一直用 `for` 和 ‵sleep` 一直跑的話，這會造成 workflow history 變很大, 當達到 server 設定的 workflow history 上限後就會出問題喔 
7. workflow 在註冊的時候只能傳入 `func` 不能傳入 `struct` 和 `pointer`

### Child workflow

​	1. 在建立 child worflow 的時候可以指定當 parent workflow 如果執行完畢的話, child workflow 是否也要跟著結束, 預設是跟著結束



### Activity

1. 每個activity 都需要是 `idempotent`
2. 如果再執行 activity 的過程中發生　`panic` 這些錯誤都將在主要的 workflow 裡面當成一般錯誤被攔截起來, 並不會直接 panic 掉然後造成workflow 無法成功執行下去
3. activity 在註冊的時候可以傳入 `func` 和 `struct`
4. 如果沒有特別指定 retrypolicy, 則當遇到錯誤的時候，系統會無限制的重試

### Distributed CRON

1. 只支援到分鐘

2. cron 語法可以參考: https://crontab.guru/

3. 每個執行都是獨立一個 workflow instance, instance, 獨立的 runID

4. 透過 dashboard 把 cron workflow terminated 之後的排程都將不在執行

5. 當 cron schedule 已經被設定了，需要先刪除再新增才能調整時間

6. 如果之前的任務還在執行, 但執行時間已經超過 cron 預設的下次觸發時間, 例如: 1分鐘, 結果是下次的觸發不會被執行, 要等上次的任務被執行完

### TCTL

1. 修改 namespace 的 retention period

   ```shell
    docker run --network=host --rm temporalio/tctl:0.29.0 --namespace {your_namespace} namespace update --retention 2
   ```

   

   
