# 3.2 Grand Central DispatchのAPI

## 3.2.1 Dispatch Queue

### Dispatch Queueとは？

[3.1.1](sect-3-1.md)で記載した開発者の役割を再度見てみると・・・

|開発者がすること                  |GCDがしてくれること                     |
|:---------------------------------|:---------------------------------------|
|実行したいタスクの定義            |必要なスレッドを生成                    |
|タスクを適切なDispatch Queueに追加|スレッドでのタスク実行をスケジューリング|

この2点を同じく先ほど記載したサンプルコードで見た場合・・・

```objectivec
dispatch_async(queue, ^{
    // 何かの処理
});
```

変数queueが__Dispatch Queue__、ブロック構文で書かれた処理が__実行したいタスク__、そしてこの2点を紐付けるのがdispatch_async関数となる。

ここで出てきたDispatch Queueとは、その名の通り処理を実行するための待ち行列(キュー)である。dispatch_async関数等により追加されたタスクは、__追加された順(First-In-First-Out, FIFO)に実行される__。

### Dispatch Queueの種類

Dispatch Queueには以下の2種類が存在する。

|Dispatch Queueの種類      |説明                             |
|:-------------------------|:--------------------------------|
|Serial Dispatch Queue     |現在実行中の処理の終了を待つ     |
|Concurrent Dispatch Queue |現在実行中の処理の終了を待たない |

__TODO:__ 図を載せる

以下のdispatch_asyncを複数実行するソースコードを実行した場合の動きをそれぞれのDispatch Queueで見てみると・・・

```objectivec
dispatch_async(queue, blk0);
dispatch_async(queue, blk1);
dispatch_async(queue, blk2);
dispatch_async(queue, blk3);
dispatch_async(queue, blk4);
dispatch_async(queue, blk5);
dispatch_async(queue, blk6);
dispatch_async(queue, blk7);
```

#### Serial Dispatch Queueの場合

Serial Dispatch Queueは現在実行中の処理を待つため、以下の通り必ず登録された順でタスクが実行される。

```objectivec
blk0
blk1
blk2
blk3
blk4
blk5
blk6
blk7
```

#### Concurrent Dispatch Queueの場合

Concurrent Dispatch Queueは現在実行中の処理を待たないため、blk0の実行が開始されるとその終了を待たずにblk1が実行されさらにその終了を待たずにblk2が実行され・・・といったように複数の処理が並列実行される。

そして、並列に実行される処理の数は以下のような現在のシステム状態に依存する。

* Dispatch Queueでの処理数
* CPUコア数
* CPU負荷

並列に実行する際は複数のスレッドを実施するが、スレッド数の決定やその管理はOS XやiOSの__XNUカーネル__が実施する。

先ほどのソースコードは、例えば以下のように複数のスレッドで実行される。

|スレッド0 |スレッド1 |スレッド2 | スレッド3 |
|:--------:|:--------:|:--------:|:---------:|
|blk0      |blk1      |blk2      |blk3       |
|blk4      |blk6      |blk5      |           |
|blk7      |          |          |           |

上記のように、Concurrent Dispatch Queueの場合は__処理の実行順番は処理内容やシステム状態により変化する__。よって、処理の実行順番を指定したい場合はSerial Dispatch Queueを使用する。

## 3.2.2 dispatch\_queue\_create



## 3.2.3 Main Dispatch Queue / Global Dispatch Queue



## 3.2.4 dispatch\_set\_target\_queue



## 3.2.5 dispatch\_after



## 3.2.6
## 3.2.7
## 3.2.8
## 3.2.9
## 3.2.10
## 3.2.11
## 3.2.12
## 3.2.13

