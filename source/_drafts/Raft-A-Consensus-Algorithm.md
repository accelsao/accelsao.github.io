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

# Majoyity Vote
處理 Split Brain 問題的方法
條件一 需要奇數的系統
條件二 多數系統的同意才可以做任何事
因為需要過半的數量 所以一但有 Split Brain 各自的一群系統也不夠數量執行任何操作
> 多數指的是**所有系統**而非只算上**活著**的系統

通常也稱作 **quorum** 系統

# Reference
https://raft.github.io/

