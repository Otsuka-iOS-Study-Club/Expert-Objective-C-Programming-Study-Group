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

図3.6はそれぞれのDispatch Queueのイメージ図である。

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

逆に、データ競合などの問題が発生しない処理を並列に実行させたい場合にはConcurrent Dispatch Queueを使用する。__なお、Concurrent Dispatch Queueについてはいくら生成してもXNUカーネルがうまく行うため、Serial Dispatch Queueのような問題は発生しない。__

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
* Concurrent Dispatch Queueを生成したい場合はDISPATCH\_QUEUE\_CONCURRENTを指定 

戻り値は、Dispatch Queueを表す「dispatch\_queue\_t型」となる。

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

Dispatch Queueは、3.2.2で説明したdispatch\_queue\_create関数を使って生成する他にも、システムが標準で提供している以下のDispatch Queueを使用することが可能。

* Main Dispatch Queue
* Global Dispatch Queue

|名前                                        |Dispatch Queueの種類      |説明                         |
|:-------------------------------------------|:-------------------------|:----------------------------|
|Main Dispatch Queue                         |Serial Dispatch Queue     | メインスレッドで実行される  |
|Global Dispatch Queue (High Priority)       |Concurrent Dispatch Queue |実行優先度: 高(最優先)       |
|Global Dispatch Queue (Default Priority)    |Concurrent Dispatch Queue |実行優先度: 標準             |
|Global Dispatch Queue (Low Priority)        |Concurrent Dispatch Queue |実行優先度: 低               |
|Global Dispatch Queue (Background Priority) |Concurrent Dispatch Queue |実行優先度: バックグラウンド |


#### Main Dispatch Queue

Main Dispatch Queueとは、以下の性質を持つDispatch Queueである。

* メインスレッドで実行される
* Serial Dispatch Queueである(メインスレッドは1つしかないため)
* このDispatch Queueに追加された処理は、メインスレッドのRunLoopで実行される
* UIの描画更新などメインスレッドでないとできない処理はこのDispatch Queueに追加して行う
* NSObjectのperformSelectorOnMainThreadインスタンスメソッドによるメソッド実行と同じ挙動

#### Global Dispatch Queue

Global Dispatch Queueとは、以下の性質を持つDispatch Queueである。

* アプリケーション全体から使用可能
* 実行優先度別に4つ存在
  - 高優先度(High Priority)
  - 標準優先度(Default Priority)
  - 低優先度(Low Priority)
  - バックグラウンド優先度(Background Priority)
* Global Dispatch Queue用にXNUカーネルで管理されるスレッドは、それぞれのGlobal Dispatch Queueの実行優先度がそのスレッドの実行優先度になる
* Global Dispatch Queue用のスレッドはXNUカーネルによりリアルタイム性が保証されているわけではないため、実行優先度はあくまで目安

#### Main Dispatch QueueとGlobal Dispatch Queueの取得方法

```objectivec
/*
 * Main Dispatch Queueの取得方法 
 */
dispatch_queue_t mainDispatchQueue = dispatch_get_main_queue();

/*
 * Global Dispatch Queue (高優先度)の取得方法 
 */
dispatch_queue_t globalDispatchQueueHigh = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);

/*
 * Global Dispatch Queue (標準優先度)の取得方法 
 */
dispatch_queue_t globalDispatchQueueDefault = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

/*
 * Global Dispatch Queue (低優先度)の取得方法 
 */
dispatch_queue_t globalDispatchQueueLow = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);

/*
 * Global Dispatch Queue (バックグラウンド優先度)の取得方法 
 */
dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
```

#### Main Dispatch QueueとGlobal Dispatch Queueのメモリ管理

Main Dispatch QueueとGlobal Dispatch Queueについては、dispatch\_retain関数やdispatch\_release関数を実行しても何も起きないし、問題も発生しない。そのため、Concurrent Dispatch Queueを生成して使用するよりもGlobal Dispatch Queueを使用したほうが簡単である。

```objectivec
/*
 * 標準優先度のGlobal Dispatch QueueでBlockを実行
 */
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{

    // 並列実行されても問題ない処理
    
    /*
     * Main Dispatch QueueでBlockを実行
     */
    dispatch_async(dispatch_get_main_queue(), ^{
        // メインスレッドでのみ実行可能な処理
    });
});
```

## 3.2.4 dispatch\_set\_target\_queue

dispatch\_queue\_create関数で生成されたDispatch Queueは、それがSerial Dispatch QueueであろうがConcurrent Dispatch Queueであろうが実行優先度は標準優先度のGlobal Dispatch Queueと同じになる。

生成したDispatch Queueの実行優先度を変更するには、dispatch\_set\_target\_gueue関数を使用する。

バックグラウンドで動く処理を実行するSerial Dispatch Queueを生成する方法は、次のソースコードの通り。

```objectivec
dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);

dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);

dispatch_set_target_queue(mySerialDispatchQueue, globalDispatchQueueBackground);
```

第1引数に実行優先度を変更したいDispatch Queueを指定し、第2引数に使用したい実行優先度と同じ優先度のGlobal Dispatch Queueを指定する。

第1引数にシステムが提供するMain Dispatch QueueやGlobal Dispatch Queueを指定すると__何が起こるかわかりません。__これらは指定してはいけません。

#### Dispatch Queueの実行階層

dispatch\_queue\_create関数によるDispatch Queueの指定は、実行優先度を変えるだけでなく、Dispatch Queueの階層構造を作ることが可能。

複数のSerial Dispatch Queueに、dispatch\_set\_target\_queue関数で、ある1つのSerial Dispatch Queueをターゲットに指定すると、並列に実行されるはずのSerial Dispatch Queueが、ターゲットのSerial Dispatch Queue上で同時に1つの処理しか実行されなくなる(図3.12)

## 3.2.5 dispatch\_after

指定した時間の経過後に処理を実行したい場合は、dispatch\_after関数を使用する。以下は3秒後に指定したBlockをMain Dispatch Queueに追加するソースコード。

```objectivec
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3ull * NSEC_PER_SEC);

dispatch_after(time, dispatch_get_main_queue(), ^{
    NSLog(@"waited at least three seconds.");
});
```

ただし、dispatch_after関数は指定した時間に処理を実行するのではなく、指定した時間にDispatch Queueに追加する点に注意。そのため厳格なタイマーとしては利用できないが、おおざっぱに処理を遅延実行させたい場合は有効。

第1引数は、時間を指定するためのdispatch\_time\_tの値。この値は、dispatch\_time関数やdispatch\_walltime関数を使用して作られる。

第2引数は、処理を追加したいDispatch Queue。

第3引数は、実行したい処理を記述したBlock。

#### dispatch\_time関数について

dispatch\_time関数は、1つ目の引数であるdispatch\_time\_t型の値で指定される時間から、2つ目の引数であるナノ秒単位で指定する時間を経過した時間を取得できる。

1つ目の引数によく使用される値として、現在の時間を表すDISPATCH\_TIME\_NOWがある。

第2引数で時間を指定する場合、数値とNSEC\_PER\_SECの積から、ナノ秒単位の数値を取得できる。（ちなみに、「ull」はC言語の数値リテラルで、型を明示する場合に使う文字列である(「unsigned long long」を表す)）。また、NSEC\_PER\_MSECを使用するとミリ秒単位にすることができる。

#### dispatch\_walltime関数について

dispatch\_walltime関数は、POSIXで使用されているstruct timespec型の時間から、dispatch\_time\_t型の値を生成する。

dispatch\_time関数は相対的な時間を作成する目的でよく使用されるが、dispatch\_walltime関数は絶対的な時間を作成する目的で使われる。

struct timespec型の時間は、NSDateクラスのオブジェクトから作成が可能。以下はNSDateクラスのオブジェクトから、dispatch\_after関数に渡すことができるdispatch\_time\_t型の値を返すソースコードである。

```objectc
dispatch_time_t getDispatchTimeByDate(NSDate *date)
{
    NSTimeInterval interval;
    double second, subsecond;
    struct timespec time;
    dispatch_time_t milestone;

    interval = [date timeIntervalSince1970];
    subsecond = modf(interval, &second);
    time.tv_sec = second;
    time.tv_nsec = subsecond * NSEC_PER_SEC;
    milestone = dispatch_walltime(&time, 0);
    return milestone;
}
```

## 3.2.6
## 3.2.7
## 3.2.8
## 3.2.9
## 3.2.10
## 3.2.11
## 3.2.12
## 3.2.13

