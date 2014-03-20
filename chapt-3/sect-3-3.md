# 3.3 Grand Central Dispatchの実装

## 3.3.1 Dispatch Queue

これまでの説明でDispatch Queueは以下の実装を含むことが予想される。

* 追加されたBlockをン管理するための、C言語レベルで実装したFIFOキュー
* Atomic関数で実装された排他制御のための軽量セマフォ
* スレッドを管理するための、C言語レベルで実装された何らかのコンテナ

#### Dispatch Queueの実装的特徴

* スレッド管理用のコードをシステムレベルで実装したもの
 * プログラマが頑張ってスレッド管理コードを実装しても勝てない
* pthreadsやNSThreadといったAPIを使うよりも効率的

#### Dispatch Queueを実現するために使用されているソフトウェアコンポーネント

|コンポーネント名                  |提供技術                      |
|:----------------------------|:---------------------------|
|libdispatch                  |Dispatch Queue              |
|Libc (pthreads)              |pthread_workqueue           |
|XNUカーネル                    |workqueue                   |

* GCD APIはlibdispatchというライブラリに含まれたC言語関数群
* Dispatch Queueは、構造体と連結リストによりFIFOキューとして実装されている
* FIFOキューはdispatch\_async関数などで追加されるBlockを管理する
* BlockはDispatch Continuationというdispatch\_continuation\_t型構造体を経てFIFOキューに入る
* Dispatch ContinuationはBlockが所属しているDispatch Groupなどを記憶するための実行コンテキストに相当

### Main Dispatch Queue

* RunLoopでBlockを実行するための仕組みであり、とりたてて面白い仕組みではない

### Global Dispatch Queue

* Global Dispatch Queue (High Priority)
* Global Dispatch Queue (Default Priority)
* Global Dispatch Queue (Low Priority)
* Global Dispatch Queue (Background Priority)
* Global Dispatch Queue (High Overcommit Priority)
* Global Dispatch Queue (Default Overcommit Priority)
* Global Dispatch Queue (Low Overcommit Priority)
* Global Dispatch Queue (Background Overcommit Priority)

Overcommit付きは、Serial Dispatch Queue要に使われる。名前の通り、システムの状態にかかわらず、強制的にスレッドを生成させるDispatch Queueになる。

### pthread\_workqueue

* Libcが提供する非公開pthreads API
* bsdthread\_registerシステムコールとworkq\_openシステムコールによってXNUカーネルのworkqueueを初期化して情報を取得する

### XNUカーネルが持つworkqueue

* WORKQUEUE\_HIGH\_PRIOQUEUE
* WORKQUEUE\_DEFAULT\_PRIOQUEUE
* WORKQUEUE\_LOW\_PRIOQUEUE
* WORKQUEUE\_BG\_PRIOQUEUE

これら4つの実行優先度は、Global Dispatch Queueのそれと同じ。

### Dispatch QueueでBlockを実行するフロー
1. libdispatchがGlobal Dispatch Queue自身のFIFOキューからDispatch Continuationを取り出し、pthread\_workqueue\_additem\_np関数を呼び出す。
2. Global Dispatch Queue自身、Global Dispatch Queueに合わせたworkqueueの情報、Dispatch Continuationを実行するためのコールバックなどを引数に渡す。
3. pthread\_workqueue\_additem\_np関数は、workq\_kernreturnシステムコールを使用して、workqueueに実行すべきアイテムが増えたことを通知する
4. XNUカーネルがシステムの状態をもとにスレッドを生成するかどうか判断する。Overcommit優先度のGlobal Dispatch Queueでは、workqueueは常にスレッドを生成します。
5. workqueueで生成されるスレッドは、workqueue要に実装されたスレッドスケジューラで動作するため通常のスレッドのコンテキストスイッチと異なる
6. workqueueスレッドは、pthread\_workqueueの関数を実行し、libdispatchのコールバック関数を呼び出す。
7. そのコールバック関数でDispatch Continuationに入ってるBlockを実行する
8. Blockの実行終了後は、Dispatch Groupへの終了通知やDispatch Continuationの開放などの】処理を実行し、Global Dispatch Queueに入っている次のBlockの実行準備にとりかかる


## 3.3.2 Dispatch Source

Dispatch SourceはBSD系カーネルではおなじみの機能「kqueue」のラッパー。

kqueue
* XNUカーネル内のイベントをハンドリングするための仕組み
* 低CPU負荷
* 低リソースk消費

### Dispatch Sourceがハンドリングするイベント

|イベント名                      |イベント内容                        |
|:-----------------------------|:------------------------------|
|DISPATCH_SOURCE_TYPE_DATA_ADD |変数が加算された                   |
|DISPATCH_SOURCE_TYPE_DATA_OR  |変数がORされた                    |
|DISPATCH_SOURCE_TYPE_MACH_SEND|MACHポートで送信した                |
|DISPATCH_SOURCE_TYPE_MACH_RECV|MACHポートで受信した                |
|DISPATCH_SOURCE_TYPE_PROC     |プロセスに関するイベントを検出した       |
|DISPATCH_SOURCE_TYPE_READ     |ファイルディスクリプタが読み込み可能になった|
|DISPATCH_SOURCE_TYPE_SIGNAL   |シグナルを受けた                    |
|DISPATCH_SOURCE_TYPE_TIMER    |タイマー                          |
|DISPATCH_SOURCE_TYPE_VNODE    |ファイルシステムに変更があった          |
|DISPATCH_SOURCE_TYPE_WRITE    |ファイルディスクリプタが書き込み可能になった |

イベント発生時には、指定したDispatch Queueで、イベント用の処理を実行できるようになっている。

### DISPATCH\_SOURCE\_TYPE\_READで非同期ファイルディスクリプタを読み込む例

```objectivec
__block size_t total = 0;
size_t size = 読み込みたいバイト数;
char *buff = (char *)malloc(size);
/*
 * 非同期ディスクリプタ（NONBLOCK）として設定
 */
fcntl(sockfd, F_SETFL, O_NONBLOCK);

/*
 * イベントハンドラを追加するためのGlobal Dispatch Queue を取得
 */
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

/*
 * READイベントでDispatch Source を作成
 */
dispatch_source_t source =
    dispatch_source_create(DISPATCH_SOURCE_TYPE_READ, sockfd, 0, queue);

/*
 * READイベント発生時に実行する処理を指定
 */
dispatch_source_set_event_handler(source, ^{

    /*
     * 読み込み可能バイト数を取得
     */
    size_t available = dispatch_source_get_data(source);

    /*
     * ディスクリプタから読み込み
     */
    int length = read(sockfd, buff, available);

    /*
     * エラー発生時はDispatch Source をキャンセル
     */
    if (length < 0) {
        /*
         * エラー処理
         */
        dispatch_source_cancel(source);
    }
    total += length;
    if (total == size) {
        /*
         * buffの処理
         */

        /*
         * 処理終了のためDispatch Source をキャンセル
         */
        dispatch_source_cancel(source);
    }
});

/*
 * Dispatch Sourceキャンセル時の処理を指定
 */
dispatch_source_set_cancel_handler(source, ^{
    free(buff);
    close(sockfd);

    /*
     * Dispatch Source（自分自身）を解放
     */
    dispatch_release(timer);
});

/*
 * Dispatch Sourceを起動
 */
dispatch_resume(source);
```

### DISPATCH\_SOURCE\_TYPE\_TIMERのタイマー例

```objectivec
/*
 * DISPATCH_SOURCE_TYPE_TIMERを指定しDispatch Source を作成。
 *
 * タイマー指定時間経過時に処理を追加するDispatch Queue としてMain Dispatch Queue を設定
 */
dispatch_source_t timer = dispatch_source_create(
    DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());

/*
 * タイマーを15 秒後に設定。
 * 繰り返しは指定しない。
 * 1秒の遅れは許容する。
 */
dispatch_source_set_timer(timer,
    dispatch_time(DISPATCH_TIME_NOW, 15ull * NSEC_PER_SEC),
        DISPATCH_TIME_FOREVER, 1ull * NSEC_PER_SEC);

/*
 * タイマー指定時間に実行する処理を指定
 */
dispatch_source_set_event_handler(timer, ^{
    NSLog(@"wakeup!");

    /*
     * Dispatch Sourceをキャンセル
     */
    dispatch_source_cancel(timer);
});

/*
 * Dispatch Sourceキャンセル時の処理を指定
 */
dispatch_source_set_cancel_handler(timer, ^{
    NSLog(@"canceled");

    /*
     * Dispatch Source（自分自身）を解放
     */
    dispatch_release(timer);
});

/*
 * Dispatch Sourceを起動
 */
dispatch_resume(timer);
```

#### Dispatch Queue
* キャンセルという概念が存在しない
 * 処理中にキャンセルを自前で実装する
 * キャンセル自体を諦める
 * NSOperationQueueなど別の方法を検討する

### Dispatch Source
* キャンセル可能
* キャンセル時コールバック処理をBlockの形で指定することが可能
* XNUカーネル内で発生するイベントハンドラの実装において、kqueueを直接使うよりもDispatch Sourceを使用する方が簡単
 * kqueueを使用しなければならない局面では、必ずDispatch Sourceを使用すること

