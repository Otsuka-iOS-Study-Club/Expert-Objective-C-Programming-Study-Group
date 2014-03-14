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

dispatch\_queue\_create関数でDispatch Queueを生成できる。

以下はSerial Dispatch Queueを生成する場合のソースコード。

```objectivec
dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);
```

#### Serial Dispatch Queueの生成数に関する注意点

Serial Dispatch Queueについては、前述のとおり以下の挙動をする。

* 複数のSerial Dispatch Queueを生成した場合、それらのQueueは並列に実行される
* Serial Dispatch Queueは、システムのリソースが許す限りいくらでも生成可能
* システムは1つのSerial Dispatch Queueに対して1つのスレッドを割り当て

つまり、Serial Dispatch Queueを大量に生成するとそれだけメモリを消費する。またコンテキストスイッチも大量に発生し、ひいてはシステムの応答性能が大幅に低下する。

よって、Serial Dispatch Queueは、複数のスレッドから同じリソースを更新するようなデータ競合などの問題を置こなさいためだけに使用するように。

* データベースの場合は、1つのテーブルに対して1つのSerial Dispatch Queueを生成
* ファイルの場合は、1つのファイルもしくは分割可能な1つのファイルブロックに対して1つのSerial Dispatch Queueを生成

逆に、データ競合などの問題が発生しない処理を並列に実行させたい場合にはConcurrent Dispatch Queueを使用する。なお、Concurrent Dispatch Queueについてはいくら生成してもXNUカーネルがうまく行うため、Serial Dispatch Queueのような問題は発生しない。

#### dispatch\_queue\_create関数について

```objectivec
dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);
```

第1引数は、

* Dispatch Queueの名前を指定
* 逆順FQDNの使用を推奨
* XcodeやInstrumentsのデバッグ中にDispatch Queue名として表示される
* アプリケーションのクラッシュ時に生成されるCrashLogにも出力される
* NULLでも良いがデバッグなどのためにちゃんとつけておきましょう

第2引数は、

* Serial Dispatch Queueを生成したい場合はNULLを指定
* Concurrent Dispatch Queueを生成したい場合はDISPATCH_QUEUE_CONCURRENTを指定 

戻り値は、Dispatch Queueを表す「dispatch_queue_t型」となる。

#### Dispatch Queueのメモリ管理

生成したDispatch Queueは明示的に開放する必要がある。なぜなら、Dispatch Queueは、BlockのようにObjective-Cのオブジェクトとして扱うための仕組みが入っていないため。

Dispatch Queueの解放は、dispatch\_release関数で行う。

```objectivec
dispatch_release(mySerialDispatchQueue);
```

dispatch\_release関数があれば、もちろんdispatch\_retain関数もある。

```objectivec
dispatch_retain(myConcurrentDispatchQueue);
```

ところで、以下のようにdispatch\_async関数でConcurrent Dispatch QueueにBlockを追加してすぐにdispatch\_release関数で解放しても大丈夫なのか？

```objectivec
dispatch_queue_t myConcurrentDispatchQueue = dispatch_queue_create("com.example.gcd.MyConcurrentDispatchQueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(myConcurrentDispatchQueue, ^{NSLog(@"block on myConcurrentDispatchQueue");});

dispatch_release(myConcurrentDispatchQueue);
```

結論として、上記の場合は問題無い。

dispatch\_async関数でDispatch QueueにBlockを追加した時点で、そのBlockがそのDispatch Queueをdispatch\_retain関数で所有することになる。Blockの実行が終了すると、そのBlockがdispatch\_release関数で所有していたDispatch Queueを解放する。

ちなみに、Dispatch Queueの他にも、「create」が名前に入っているAPIはその生成したものが不要になった場合はdispatch\_release関数で解放する必要がある。また、関数やメソッドで生成されたものを受け取った場合はdispatch\_retain関数で所有する必要がある。

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

