---
title: "平行化你的 Python 程式"
date: 2021-04-14T15:27:23+08:00
draft: false
categories:
- Web
- Python
tags:
- Web
- Python
- Parallel
---

在運行程式時，我們常希望善用系統的多核環境平行處理資料以提升計算的效率。然而 Python 的平行化似乎沒有那麼簡單。這篇文章會簡單介紹平行運算的概念、平行化 Python 時會遇到的問題、以及如何解決它們。

![](https://i.imgur.com/3MDT9fN.png)
<!--more-->

## Multiprocessing vs. Multithreading
Multiprocessing 是一種將程式或指令同時跑在**多個平行 process** 的運算模式。當你的工作可以高度平行化時，也就是 subtask 之間是互相獨立的情況下，multiprocessing 就可以利用多核的環境幫助你加速運算。

說到 multiprocessing 就不得不提到 multithreading。Multithreading 與 multiprocessing 很相似，目標都是要將工作平行化處理，然而在 multithreading 中，多個平行的 thread **共享 heap (動態記憶體空間)、data section (global/staic 變數)、與一些 OS 資源**，如 open files 與 signals。因此若多個 thread 共享某塊 memory 內的資料，那麼程式設計者就必須使用一系列 synchronization 的技巧來預防 dead lock 與 race condition。這是一個很大的主題，在這邊先不深究，以後有機會會寫一篇文章另外講解。

## Python 與平行化
CPython 有一個 GIL (Global Interpreter Lock) 的設計，它保證在任何時刻只會有一個 thread 執行 Python code 以確保 multithreading 下的 thread safety。當**執行一定數量的 code 或是 I/O 被阻塞**時， CPython 為了平衡不同 thread 的執行時間，會強制釋放 GIL 並轉移到其他的 thread 上。

在 single thread 的情況下 GIL 不會造成嚴重的問題。但是當我們嘗試把工作分佈到多個 thread 上時，GIL 會 CPU-bound 的工作效率低下，這是因為 GIL 限制了多個 thread 的並行。要注意的是，GIL 不會對 IO-bound task 造成影響，這是因為 GIL 在 thread 等待 I/O 時就會被釋放。

然而在資料處理上，大部分的 task 都還是 CPU-Bound 的，難道 Python 就不能平行話 CPU-bound 的工作了嗎？ 當然不是。我們還有 multiprocessing 可以使用！當我們運行多個 Python 的 process 時，每個 Process 都會有一個它自己的 Python interpreter 與記憶體空間，這解決了上述提到的 GIL 的問題。雖然運行多個 process 的開銷遠比 thread 還要大，但對於 CPU-bound task 來說這樣的 overhead 是可以忽略的。

接下來會展示如何以 Python 的 `multiprocessing` 套件實作 worker pool，也就是維護數個運行的 worker，每個 worker 都是一個 process，並將 task 切成數個 subtask 分配給這群 worker。當一個 worker 完成當前工作後，它會去檢查還有沒有尚未被其他 worker 執行的 subtask，若有則拿去執行。

Worker pool 讓我們能夠節省系統資源的開銷。如果我們每新增一個 subtask 就創建一個 process，不僅 overhead 很大而且也會很快就耗盡系統資源。
## 實作 Worker Pool
假設 `demo` 函式是我們希望平行運行的 subtask，它簡單的將兩個整數相加並印出結果：
```python
def demo(a: int, b: int):
    print(f'{a} + {b} = {a+b}')
    return a + b
```
使用當前 cpu 數量做為 pool size 並行處理 `demo(0, 0)`, `demo(1, 1)` ... `demo(9, 9)`。`starmap()` 方法讓我們傳入需要的參數並阻塞等待所有 subtask 完成，最後在結束時返回所有運行的結果：
```python
from multiprocessing import Pool
from multiprocessing import cpu_count

def pool_sync():
    pool_sz = cpu_count()
    with Pool(pool_sz) as p:
        res = p.starmap(demo, [(i, i) for i in range(10)])
        p.close()
        p.join()

    print(res)
```
使用 `starmap_async` 會馬上返回並回傳一個 callback。我們可以用 `callback.get()` 來取得所有運行結果：
```python
def pool_async():
    pool_sz = cpu_count()
    with Pool(pool_sz) as p:
        # returns immediately
        cb = p.starmap_async(demo, [(i, i) for i in range(10)])
        res = cb.get()
        p.close()
        p.join()

    print(res)
```
完整程式碼：
```python
from multiprocessing import Pool
from multiprocessing import cpu_count

def demo(a: int, b: int):
    print(f'{a} + {b} = {a+b}')
    return a + b

def pool_sync():
    pool_sz = cpu_count()
    with Pool(pool_sz) as p:
        res = p.starmap(demo, [(i, i) for i in range(10)])
        p.close()
        p.join()

    print(res)

def pool_async():
    pool_sz = cpu_count()
    with Pool(pool_sz) as p:
        # returns immediately
        cb = p.starmap_async(demo, [(i, i) for i in range(10)])
        res = cb.get()
        p.close()
        p.join()

    print(res)

if __name__ == '__main__':
    pool_sync()
    pool_async()
```
可能的 output：
```
0 + 0 = 0
1 + 1 = 2
2 + 2 = 4
3 + 3 = 6
4 + 4 = 8
5 + 5 = 10
7 + 7 = 14
6 + 6 = 12
8 + 8 = 16
9 + 9 = 18
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
0 + 0 = 0
1 + 1 = 2
2 + 2 = 4
3 + 3 = 6
5 + 5 = 10
4 + 4 = 8
6 + 6 = 12
9 + 9 = 18
7 + 7 = 14
8 + 8 = 16
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```
注意到每個 subtask 並不是按照順序而是平行的執行，但最後返回的結果是按照我們輸入給 `starmap` 或是 `starmap_async` 的參數順序的。
## 總結
在資料科學的領域中，我們常常需要從資料庫撈出海量資料、預處理後再做進一步的分析。如果能善用資料平行處理的技巧可以大大提高運算的效率。希望大家看完這篇文章後能更了解 multiprocessing/multithreading 與 GIL 的概念，並且能實際應用在 data preprocessing pipeline 中！