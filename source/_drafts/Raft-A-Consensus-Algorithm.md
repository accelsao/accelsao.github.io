---
title: Raft - A Consensus Algorithm
tags:
- raft
- consensus algorithm
---

# Raft 是什麼?
Raft 被設計的原因是 Paxos 共識算法太過困難 (非常)
Raft 相較之下淺顯易懂 而且 paper 詳細的說明**實作方法**
效率的也不比 Paxos 差

以上兩者都是容錯分布式系統

<!--more-->

## 其他的容錯系統
* MapReduce 複製計算但需要一個 master 維護
* GFS 複製資料但需要 master 來選擇 primary
* VMware FT 複製服務端但需要 test-and-set 來選擇 primary

以上都用一個系統來處理 所以不會有 split brain 的問題

# Split Brain
Split Brain 發生情況
假設我們現在複製 test-and-set service
現在有這些系統 [C1, C2, S1, S2]
假設 Client 1 C1 要求設 x 為 1, Server 回傳原本的狀態 但只有一個 Client 可以得到因為有鎖
如果 C1 只能和 S1 溝通 但 S2 無法 那 C1 該繼續嗎?
如果 S2 當機了 那**只能**繼續執行 因為這就是容錯系統的目的
如果 S2 還在但網路掛掉 C1 照理說不該繼續 因為 S2 可能和 C2 在溝通

於是乎 要不 C1 繼續執行 但容錯系統沒有意義 要不 C1 不執行 結果 C1-S1, C2-S2 各自分裂成一組形成 Split Brain

最主要的問題就是無法分辨 **系統當機** 還是 **網路壞掉** 兩者一樣問題 都拿不到回傳的訊息

# 多數決 Majority Vote
處理 Split Brain 問題的方法
條件一 需要奇數的系統
條件二 多數系統的同意才可以做任何事
因為需要過半的數量 所以一但有 Split Brain 各自的一群系統也不夠數量執行任何操作
> 多數指的是**所有系統**而非只算上**活著**的系統

通常也稱作 **quorum** 系統


# Raft 簡述
Client 給 Leader Put/Get 指令
Leader 接收指令增加到 Log
Leader 傳送 AppendEntries 給其他 Follower
Followers 接收指令增加到 Log
(每個人都有自己的 Log )
Leaders 這時候會等待其他人的回覆 只要包含自己超過半數 就會 Commit
Commit 過後的操作就會被"永久"保存 就算 Fail 了之後重啟依然要有 Commit 後的結果
Majority 在下一個 Leader 的 Votes Request 就會知道 Commited Log
Leader 執行 Command 回傳給 Client
Leader "piggyback" commit info in next AppendEntries
Followers 就會執行 commited logs

## 為什麼需要 Log Key/Value DataBase 不夠嗎?
Log 保有 Command 的順序, Leader 可以比對 Follower 的 Log
Log 保存暫時性的 Commands, 也有永久性的 能夠在 reboot 後得到遺失的 Log

## Server Log 會剛好和 Replicas 一樣嗎?
1. Replicas 有延遲
2. 很多問題會導致它們的 logs 長的不同
不過 根據一些規則他們會調整成一樣的 log 而且 commit 的機制保證只會處理正確的 log

Raft 有兩個重要的設計
1. Leader 的選舉
2. 系統重啟後保證 log 相同

## 為什麼需要 Leader?
為了保證 Replicas 都做一樣的指令 不是所有設計都有 Leader 例如 Paxos 就沒有
對於一個新的 Leader 都會有一個新的數字(term)
每個數字至多一個 Leader (可能0個)
這個數字非常重要用來確保我們只聽最新的 Leader 處理指令 避免衝突

## 開始選舉的時間點?
每個系統會等待一段時間 (election timeout) 如果這時候都沒有收到其他 Leader 的訊息
系統就會成為 Candidate(候選人) 然後開始向其他系統**收票**
當然這時候舊的 Leader 仍然會認為自己是 Leader 所以系統成為 Candidate 後會將用最新的數字(term) 和其他人拜票
這時候的 term 就發揮的重大的作用

## 怎麼保證 Leader 只有一個?
很簡單 我們只有拿到過半數的時候才會升等為 Leader 而既然要過半數 那就最多只有一個

## 怎麼知道選出 Leader 了沒?
新的 Leader 產生後就會向其他人傳送 **heart-beats** 來表示 Leader 的存在
當收到別人的 **heart-beats** election timeout 就會重新計時 如此就不會開始選舉階段

## 選舉失敗的原因
1. 剩下的人數沒有過半
2. 剛好 選票一直都分散 (split vote)
Raft 如何避免 split vote?
每個人的 election timeout 都是隨機的時間 有長有短 基本上時間一旦錯開 就很難有 split vote 的發生 (這是很常見的策略)

## election time 如何選?
1. 時間要夠長 讓 Candidate 能夠蒐集到選票並且成為 Leader
2. 因為網路問題 heart-beats 可能送不到 所以至少有幾個 heart-beats 間隔
3. 時間也不能太長 如果有 failure 可以及時應對 不會停擺在那邊

## 如果舊 Leader 不知道已經有新的 Leader 呢?
可能舊 Leader 沒收到選舉的 message
可能他只收到少數人的 不過也因此他不會 commit 或 execute 任何指令 但是他的 log 會被更新 所以長得和新 Leader 會不同

# Raft Log
當 Leader 產生後
1. Client 只能和 Leader 溝通
2. Client 見不到 Followers 的 logs
當 Leader 換人後
1. 怎麼處理不同的 replicas
2. 怎麼處理漏掉的指令
3. 怎麼處理重複的指令

我們需要確保哪些事情?
如果 server 在某個 log entry 執行了某個指令, 那就要確保這個 log entry 不會執行其他的指令
Why?
如果 server 否決了指令 結果 Leader 換了一個  對 Client 來說他說見到的情況可能就會不同 那麼就違反了 Single Server 的目標
舉個例子
S1: put(k1,v1), put(k1,v2)
S2: put(k1,v1), put(k2,x)
第二個操作這兩個 Server 都不能執行

log 是怎麼產生分歧的?
在 leader 送給其他人 AppendEntries 之前就發生 crash
S1: 3
S2: 3, 3
S3: 3, 3
> S1 還沒收到第 2nd log entry 的消息

最壞的情況甚至同個 log entry 有不同的值
S1: 3
S2: 3, 3, 4
S3: 3, 3, 5

Raft 的 Follower 會採用新的 Leader 的 log, 用來保證 log enrty 的一致性

這是 AppendEntries 的規則 (論文 Figure2)
![](https://i.imgur.com/kp49L2u.png)


|log entry|10|11|12|13|
|---|---|---|---|---|
|S1   |3   |    |    |   |
|S2   |3   |3   |4   |   |
|S3   |3   |3   |5   |   |


1. S3 成為 Leader, Term = 6
2. S3 傳送 AppendEntries ,log enrty 13, prevLogIndex=12, prevLogTerm=5
3. S2 根據 Rule2 他 log entry 的 term 是 4 不是 5 所以會回傳 False
4. S3 傳送 AppendEntries **這次會將上個發生衝突的 log entry 一起傳送**, log enrty 12+13, prevLogIndex=11, prevLogTerm=3
5. S2 根據 Rule3 因為他們的 log enrty 11 都有 term = 3 所以 S2 會把 log enrty 12+13 丟棄然後覆蓋成 S3 的 log entry 12+13

這種處理方式可以讓 log 和 Leader 不同的 Follower 得到正確的 log

不過要注意到我們只能把**尚未提交**的 log 給覆蓋掉 否則就會出事 --- 已經告訴 Client 的指令卻不作數

為什麼不把有**最長** log 的 server 當作 Leader?

|log entry|10|11|12|
|---|---|---|---|---|
|S1   |5   |6	|  7|
|S2   |5   |8   |   |
|S3   |5   |8   |   |

首先 這個情況可能嗎? 
首先 S1 在 term 6 的時候 當機後重啟 在 term 7 的時候 又當機 兩次都只有處理到自己的 logs
在 S1 重啟之前 S2 當了 Leader 並且 term = 8 因為 S2 or S3 至少有一個在 S1 拜票的時候就知道 term 是 7, 所以接下來從 8 開始
S1 都還沒重啟 只有 S2 和 S3 都得到 term 8 的 log, 然後都當機
假設現在 3 個 server 都重啟 那換誰當 Leader?
S1 有最長的 Log 問題是 term 8 可以已經**提交**了 我們只能讓 S2 or S3 當 leader,  所以用 log 長度來決定 Leader 會出事

5.4.1 Election restriction 提到了選舉的限制
投票對象只能是 "as least as up to date", 候選人的最後一個 log entry 有最新的 term, 或者 候選人最後的 term 一樣且 log 長度至少等長
所以 S2, S3 不會投給 S1, 只會互相給對方, S1 的 6 和 7 因為也不是**多數** 所以也不可能提交給 Client, 所以丟棄後也不會造成問題
"as least as up to date" 的規則確保 Leader 有所有提交過的 log, 所以不會把提交過的 log 給丟棄

# Figure 7
![](https://i.imgur.com/Rff7e8S.png)

# FAQ



# Reference
https://raft.github.io/

