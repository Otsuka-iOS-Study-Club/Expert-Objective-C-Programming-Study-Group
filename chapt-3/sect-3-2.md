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

## 3.2.6 Dispatch Group

Dispatch Queueに追加した複数の処理がすべて終了してから終了処理を実行したい場合には、Dispatch Groupを使用する。

Global Dispatch Queueに3つBlockを追加し、それらのBlockの実行がすべて終了したらMain Dispatch Queueで終了処理用のBlockが実行される例は以下のようになる。

```objectivec
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{NSLog(@"blk0");});
dispatch_group_async(group, queue, ^{NSLog(@"blk1");});
dispatch_group_async(group, queue, ^{NSLog(@"blk2");});

dispatch_group_notify(group,
    dispatch_get_main_queue(), ^{NSLog(@"done");});
dispatch_release(group);
```

このソースコードの実行結果は次のようになる。

```objectivec
blk1
blk2
blk0
done
```

blk0~2は複数スレッド上で並列に実行されるが、「done」は必ず最後に出力される。

Dispatch Group を使うことで、すべての並列処理の実行終了を監視することができる。そしてすべての処理の実行終了を検知したら、Dispatch Queue に終了処理を追加することができる。

### dispatch\_group\_create

dispatch\_group\_t型のDispatch Groupを生成する。

dispatch\_group\_createによって生成されたDispatch Groupは **dispatch\_release関数で解放する必要がある。** 

### dispatch\_group\_async

dispatch\_async関数と同じく、指定したDispatch QueueにBlockを追加する。

dispatch\_async関数との違いは、第1引数に生成したDispatch Groupを指定する点。

第2引数で指定されたBlockは、第1引数で指定されたDispatch Groupに所属することになる。

BlockをDispatch Groupに所属させると、そのBlockはそのDispatch Groupをdispatch\_retain関数で所有することになり、Blockの実行が終了するとdispatch_release関数で所有していたDispatch Groupを解放する。

-> **Dispatch Groupを使い終わったらすぐさまdispatch\_release関数で開放してかまわない、ということ。**

### dispatch\_group\_notify

Dispatch Groupに追加された処理が全て実行終了した後に実行されるBlockをDispatch Queueに追加する。

第1引数には監視したいDispatch Group、第3引数には全処理実行終了後に実行されるBlock、第2引数にはそのBlockを追加するDispatch Queueを指定する。

### dispatch\_group\_wait

Dispatch Groupに所属させた全ての処理が終了するまで待機する　(同期処理)。

第1引数には処理待ち対象のDispatch Group、第2引数にはいつまで待つかの時間 (タイムアウト) を指定する (dispatch\_time\_t型)。

下記ソースコードのようにDISPATCH_TIME_FOREVERを使用するとDispatch Groupに所属させた全ての処理が終了するまで永遠に待つことを意味する。途中でのキャンセル不可。

```objectivec
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{NSLog(@"blk0");});
dispatch_group_async(group, queue, ^{NSLog(@"blk1");});
dispatch_group_async(group, queue, ^{NSLog(@"blk2");});

dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
dispatch_release(group);
```

dispatch\_after関数のように1秒間待つには次のように指定する。

```objectivec
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
long result = dispatch_group_wait(group, time);
if (result == 0) {
    /*
     * Dispatch Groupに所属させたすべての処理の実行が終了した
     */
} else {
    /*
     * Dispatch Groupに所属させた処理のいずれかが、まだ実行中
     */
}
```

dispatch\_group\_wait関数の戻り値が0以外の場合、指定した時間経過後もDispatch Groupに所属させた処理が実行中であることを意味する。

すなわちDISPATCH\_TIME\_FOREVERを指定した場合は必ず0が戻る。

DISPATCH\_TIME\_NOWを指定するとDispatch Groupに所属させた処理の実行終了を一切待たずに判定することができる。

```objectivec
long result = dispatch_group_wait(group, DISPATCH_TIME_NOW);
```

これによりメインスレッドのRunLoopのループごとに、余計な待ち時間を作らずに実行終了しているか否かのチェックが可能となる。

しかしその場合はdispatch\_group\_notify関数でMain Dispatch Queueに終了処理を追加する方がより簡潔になるのでオススメ。


## 3.2.7 dispatch\_barrier\_async

3.2.1で前述したとおり、DBやファイルへのアクセスの際にSerial Dispatch Queueを使うことでデータの強豪を回避することが可能となる。

その上、読み込み処理にConcurrent Dispatch Queueを使い、書き込み処理に読み込み処理が実行されていない状態でSerial Dispatch Queueを使用することでより効率的にアクセスすることが可能となる。

GCDでは、そのような処理制御をよりスマートに行うために dispatch\_barrier\_async関数を用意している。

この関数は、dispatch\_queue\_create関数で生成したConcurrent Dispatch Queueとともに使用する。

以下に例を示す。

```objectivec
dispatch_queue_t queue = dispatch_queue_create(
    "com.example.gcd.ForBarrier", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(queue, blk0_for_reading);
dispatch_async(queue, blk1_for_reading);
dispatch_async(queue, blk2_for_reading);
dispatch_async(queue, blk3_for_reading);
dispatch_async(queue, blk4_for_reading);
dispatch_async(queue, blk5_for_reading);
dispatch_async(queue, blk6_for_reading);
dispatch_async(queue, blk7_for_reading);

dispatch_release(queue);
```

上記処理の内、blk3\_for\_readingとblk4\_for\_readingの間で書き込み処理を実行し、blk4\_for\_reading以降では書き込み後の内容を読み込ませる場合は以下のようにする。

```objectivec
...
dispatch_async(queue, blk0_for_reading);
dispatch_async(queue, blk1_for_reading);
dispatch_async(queue, blk2_for_reading);
dispatch_async(queue, blk3_for_reading);
dispatch_barrier_async(queue, blk_for_writing);
dispatch_async(queue, blk4_for_reading);
dispatch_async(queue, blk5_for_reading);
dispatch_async(queue, blk6_for_reading);
dispatch_async(queue, blk7_for_reading);
...
```

この場合dispatch\_barrier\_async関数は、Concurrent Dispatch Queueに追加されたblk0\_for\_reading~blk3\_for\_readingがすべて実行終了してから、blk\_for\_writingをConcurrent Dispatch QUeueに追加する。

そしてblk\_for\_writingが実行終了したあと、block4\_for\_reading以降を通常のConcurrent Dispatch Queueの動作で実行する。

__Concurrent Dispatch Queueとdispatch\_barrier\_async関数を使って、効率の良いDB、ファイルアクセスを実装しましょう。__ 


## 3.2.8 dispatch\_sync

dispatch\_asyncっ関数は **指定されたBlockを、指定されたDispatch Queueに「非同期処理」として追加する。** 

一方、指定されたBlockを、指定されたDispatch Queueに「同期処理」として追加するにはdispatch\_sync関数を使用する。

```objectivec
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
	
dispatch_sync(queue, ^{/* 処理*/});
```

dispatch\_sync関数を呼び出すと、指定した処理の実行が終了するまで、dispatch]}sync関数から返ってこなくなるため、簡易版dispatch\_group\_wait関数と言える。

dispatch\_sync関数は簡単に使える一方で簡単に __デッドロック__ を引き起こす恐れがある。

以下に例を示す。

```objectivec
dispatch_queue_t queue = dispatch_get_main_queue();
dispatch_sync(queue, ^{NSLog(@"Hello?");});
```

このコードはメインスレッドで動く予定のBlockの実行待ちをする。これをメインスレッドで実行すると、Main Dispatch Queueに追加されたBlockが実行できなくなるためデッドロックに陥る。

次の例も同様にデッドロックを引き起こす。

```objectivec
dispatch_queue_t queue = dispatch_get_main_queue();
dispatch_async(queue, ^{
    dispatch_sync(queue, ^{NSLog(@"Hello?");});
});
```

Main Dispatch Queueで実行されるBlockが、Main Dispatch Queueに実行させたいBlock ("Hello?"出力処理) の実行終了を待つ。 絵に描いたようなデッドロックである。

Serial Dispatch Queueでも同様のことが起こる。

```objectivec
dispatch_queue_t queue =
    dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);
dispatch_async(queue, ^{
    dispatch_sync(queue, ^{NSLog(@"Hello?");});
});
```

なお、dispatch\_barrier\_async関数にasyncと入っているとおり、dispatch\_barrier\_sync関数も存在する。

dispatch\_barrier\_async関数の効力である、それまでに追加されていた処理の実行終了をまってからDispatch Queueに処理を追加することに加えて、dispatch\_sync関数と同じく、その追加した処理の実行終了も待ちます。

__dispatch\_synch関数など、処理の実行を同期で待つAPIはデッドロックのリスクを考慮して使用することが大切である。__


## 3.2.9 dispatch\_apply

dispatch\_apply関数は、 **指定した回数分、指定したBlockを、指定したDispatch Queueに追加し、それらすべての処理が終了するまで待つAPIである。** 

```objectivec
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply(10, queue, ^(size_t index) {
    NSLog(@"%zu", index);
});
NSLog(@"done");
```

このソースコードの実行結果は次のようになる。

```objectivec
4
1
0
3
5
2
6
8
9
7
done
```

Global Dispatch Queueで処理を実行しているため、それぞれの処理実行タイミングは一定ではないが、dispatch\_apply関数がすべての処理実行終了を待つことによって「done」は必ず最後に出力される。

第1引数は繰り返し回数、第2引数は追加対象Dispatch Queue、第3引数は追加する処理 (Block) である。

今までの例とは異なり、第3引数のBlockは引数付きのBlockになっている。これは、第1引数の回数分繰り返しBlockを追加するため、それぞれのBlockを区別するために使用される。

このためNSArrayクラスオブジェクトのすべてのエントリに対して処理実行を行いたい場合、forループ文を書かなくてもよくなる。

```objectivec
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply([array count], queue, ^(size_t index) {
    NSLog(@"%zu: %@", index, [array objectAtIndex:index]);
});
```

なお、dispatch\_apply関数もdispatch\_sync関数同様に処理の実行終了を待つため、dispatch\_apply関数をdispatch\_async関数で非同期実行することをオススメする。

```objectivec
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
/*
 * Global Dispatch Queueで非同期実行
 */
dispatch_async(queue, ^{
    /*
     * Global Dispatch Queue上で、
     * dispatch_apply関数での処理の実行終了をすべて待つ
     */
    dispatch_apply([array count], queue, ^(size_t index) {
        /*
         * NSArrayオブジェクトに含まれるすべてのオブジェクトを並列処理
         */
        NSLog(@"%zu: %@", index, [array objectAtIndex:index]);
    });

    /*
     * dispatch_apply関数での処理がすべて実行終了
     */

	/*
     * Main Dispatch Queueで非同期実行
     */
    dispatch_async(dispatch_get_main_queue(), ^{
        /*
         * Main Dispatch Queueで処理実行
         * ユーザーインターフェース更新など
         */
        NSLog(@"done");
    });
});
```


## 3.2.10 dispatch\_suspent / dispatch\_resume

dispatch\_suspend関数で指定したDispatch Queueをサスペンドさせ、dispatch\_resume関数でレジュームさせることができる。

```objectivec
dispatch_suspend(queue);
dispatch_resume(queue);
```

これらの関数はすでに実行されている処理には影響しない。

サスペンドはDispatch Queueに追加されているが未実行な処理をこれ以降実行させなくし、レジュームはサスペンドしたものをまた実行できるように戻す。


## 3.2.11 Dispatch Semaphore

並列処理におけるデータ不整合を回避するためには **Serial Dispatch Queueやdispatch\_barrier\_async** を使う。

ではより粒度の細かな排他制御が必要な場合はどうすればいいのか？

以下の例は、その順番は気にしないが、すべてのデータをNSMutableArrayに追加したい場合。

```objectivec
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
NSMutableArray *array = [[NSMutableArray alloc] init];
for (int i = 0; i < 100000; ++i) {
    dispatch_async(queue, ^{
        [array addObject:[NSNumber numberWithInt:i]];
    });
}
```

このソースコードは実行すると、高確率でメモリ関連のエラーによりアプリケーションが不正終了する。

こんなときに役立つのが、そう、 **Dispatch Semaphore。** 
* 計数型セマフォ (カウンタを持ったセマフォ)
* カウンタ0の時は待ち
* カウンタ1以上のときは1減算して待たずに進む

### dispatch\_semaphore\_create
* Dispatch Semaphoreを生成
* 引数： カウンタの初期値
* dispatch_release関数で要解放
* dispatch_retain関数による所有も可能

```objectivec
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
```

### dispatch\_semaphore\_wait
* Dispatch Semaphoreのカウンタが1以上になるまで待つ
* カウンタが1以上になったとき関数から返る
* 第2引数： dispatch_time_t型の値による待ち時間
* 戻り値： dispatch_group_wait関数と同様に0 (処理実行終了) or others (処理実行未終了)

```objectivec
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```

一例として以下のようになる。

```objectivec
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
long result = dispatch_semaphore_wait(semaphore, time);
if (result == 0) {
    /*
     * Dispatch Semaphoreのカウンタが1 以上だったため、
     * もしくは、指定時間待機中に、
     * Dispatch Semaphoreのカウンタが1 以上になったため、
     * Dispatch Semaphoreのカウンタを1 減算した。
     *
	 * 排他制御が必要な処理を実行可能
     */
} else {
    /*
     * Dispatch Semaphoreのカウンタが0 だったため、
     * 指定時間経過まで待機した
     */
}
```

冒頭のソースコードでDispatch Semaphoreを用いると、

```objectivec
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
/*
* Dispatch Semaphoreを生成。
*
* Dispatch Semaphoreのカウンタ初期値を「1」に設定。
*
* NSMutableArrayクラスのオブジェクトにアクセスできる
* スレッドは同時に1 つしかないことを保証する
*/
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
NSMutableArray *array = [[NSMutableArray alloc] init];
for (int i = 0; i < 100000; ++i) {
    dispatch_async(queue, ^{
        /*
         * Dispatch Semaphoreを待機。
         *
         * Dispatch Semaphoreのカウンタが1 以上になるまで永遠に待つ
         */
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        
		/*
         * Dispatch Semaphoreのカウンタが1 以上になったため、
         * Dispatch Semaphoreのカウンタを1 減算して、
         * dispatch_semaphore_wait関数から実行が戻った状態。
         *
	     * つまり、ここが実行されているときの
         * Dispatch Semaphoreのカウンタは常に「0」。
         *
         * NSMutableArrayクラスのオブジェクトに
         * アクセスできるスレッドは1 つだけなので
         * 安全に更新できる。
         */
        [array addObject:[NSNumber numberWithInt:i]];
        
		/*
         * 排他制御したい処理が終わったので、
         * dispatch_semaphore_signal関数で
         * Dispatch Semaphoreのカウンタを1 加算。
         *
         * dispatch_semaphore_wait関数で
         * Dispatch Semaphoreのカウンタの加算を待っているスレッドがいれば、
         * そのうちの一番最初から待っていたスレッドが動き出す。
         */
        dispatch_semaphore_signal(semaphore);
    });
}
/*
 * 使い終わったら、本来は次のように
 * Dispatch Semaphoreを解放する必要がある
 *
 * dispatch_release(semaphore);
 */
```

ごく一部だけ排他制御が必要な状況でDispatch Semaphoreは威力を発揮する。


## 3.2.12 dispatch\_once

指定した処理をアプリケーション実行中に1度だけ実行することを保証するためのAPI。

以下のようなよくある初期化処理は、

```objectivec
static int initialized = NO;
if (initialized == NO)
{
    /*
     * 初期化
     */
    initialized = YES;
}
```

dispatch\_once関数を使うことで以下のようになる。

```objectivec
static dispatch_once_t pred;
dispatch_once(&pred, ^{
    /*
     * 初期化
     */
});
```

dispatch\_once関数を使うと、このような初期化処理がマルチスレッド環境下で実行されたとしても、完全に安全であることが保証できる。

特にシングルトンパターンでの、シングルトンオブジェクトを生成する際に有効活用できる。


## 3.2.13 Dispatch I/O

Dispatch I/OとDispatch Dataを用いることで、サイズの大きいファイルの読み込みを複数スレッドによる並列読み込みが可能となる。

1つのファイルを、あるサイズごとにGlobal Dispatch Queueを使用してread/writeする。

```objectivec
dispatch_async(queue, ^{/* 0～ 8191 バイトまで読み込む処理*/});
dispatch_async(queue, ^{/* 8192～16383 バイトまで読み込む処理*/});
dispatch_async(queue, ^{/* 16384～24575 バイトまで読み込む処理*/});
dispatch_async(queue, ^{/* 24576～32767 バイトまで読み込む処理*/});
dispatch_async(queue, ^{/* 32768～40959 バイトまで読み込む処理*/});
dispatch_async(queue, ^{/* 40960～49151 バイトまで読み込む処理*/});
dispatch_async(queue, ^{/* 49152～57343 バイトまで読み込む処理*/});
dispatch_async(queue, ^{/* 57344～65535 バイトまで読み込む処理*/});
```

* AppleによるDispatch I/OとDispatch Dataの使用例 (Apple System Log APIのソースコードから抜粋)

```objectivec
pipe_q = dispatch_queue_create("PipeQ", NULL);
pipe_channel = dispatch_io_create(DISPATCH_IO_STREAM, fd, pipe_q, ^(int err){
    close(fd);
});

*out_fd = fdpair[1];
dispatch_io_set_low_water(pipe_channel, SIZE_MAX);
dispatch_io_read(pipe_channel, 0, SIZE_MAX, pipe_q,
　　　　　　　　^(bool done, dispatch_data_t pipedata, int err){
　　　　if (err == 0)
　　　　{
　　　　　　　　size_t len = dispatch_data_get_size(pipedata);
　　　　　　　　if (len > 0)
　　　　　　　　{
　　　　　　　　　　　　const char *bytes = NULL;
　　　　　　　　　　　　char *encoded;
　　　　　　　　　　　　
　　　　　　　　　　　　dispatch_data_t md = dispatch_data_create_map(
　　　　　　　　　　　　　　　　pipedata, (const void **)&bytes, &len);
　　　　　　　　　　　　encoded = asl_core_encode_buffer(bytes, len);
　　　　　　　　　　　　asl_set((aslmsg)merged_msg, ASL_KEY_AUX_DATA, encoded);
　　　　　　　　　　　　free(encoded);
　　　　　　　　　　　　_asl_send_message(NULL, merged_msg, -1, NULL);
　　　　　　　　　　　　asl_msg_release(merged_msg);
　　　　　　　　　　　　dispatch_release(md);
　　　　　　　　}
　　　　}

　　　　if (done)
　　　　{
　　　　　　　　dispatch_semaphore_signal(sem);
　　　　　　　　dispatch_release(pipe_channel);
　　　　　　　　dispatch_release(pipe_q);
　　　　}
});
```

### dispatch_io_create
* Diapatch I/Oを生成
* 第1引数： Dispatch I/Oチャネルタイプ (DISPATCH_IO_STREAM / DISPATCH_IO_RANDOM)
* 第2引数： ファイルディスクリプタ
* 第3引数： 第4引数のBlockを実行するDispatch Queue
* 第4引数： エラー発生時に実行される処理を実行するBlock

### dispatch_io_set_low_water
* 一度に読み込むサイズ (分割サイズ) を設定
* 第1引数： dispatch_io_createで生成したDispatch I/Oチャネル
* 第2引数: 最小読み込みサイズ (bytes)。すべてのデータを読み込みたい場合は **SIZE_MAX** を指定する

### dispatch_io_read
* Global Dispatch Queueを使って並列読み込みを実行
* 第1引数: dispatch_io_createで生成したDispatch I/Oチャネル
* 第2引数: オフセットサイズ。ランダムアクセスチャネルタイプの際にのみ使用するため、ストリームベースチャネルの場合は無視される
* 第3引数: 読み込むファイルサイズ。EOFまで読み込む場合は **SIZE_MAX** を指定する
* 第4引数: 第5引数のBlockを実行するDispatch Queue
* 第5引数: 読み込み終了コールバック用Block

### dispatch_io_handler_t
* Dispatch I/Oチャネル上で使用されるハンドラBlock (dispatch_io_read関数の第5引数)
* 第1引数: 処理完了判定フラグ
* 第2引数: 処理されたデータオブジェクト。上記例では読み込まれたファイルデータ
* 第3引数: エラーコード

このようにDispatch I/Oを使用することで通常よりも高速にファイル読み込みを実装することができる。
