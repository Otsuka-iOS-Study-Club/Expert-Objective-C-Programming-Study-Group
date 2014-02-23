# 1.4 ARCの実装

## 1.4.1 __strong修飾子

### __strong修飾子のついた変数への代入 (自分が生成し、所有する場合)
* 例

```objectivec
	// オリジナル
    {
        id __strong obj = [[NSObject alloc] init];
    }
```

```objectivec
    /* コンパイラによる擬似コード */
    id obj = objc_msgSend(NSObject, @selector(alloc));
    objc_msgSend(obj, @selector(init));
    objc_release(obj);
```

1. NSObjectのalloc, initメソッド呼び出し
2. スコープを抜けるのでrelease呼び出し

### __strong修飾子のついた変数への代入 (自分が生成・所有しない場合)
* 自分が生成・所有しない、とは？？  
 alloc/new/copy/mutableCopyメソッド以外のメソッドを使ってオブジェクトを取得すること。ARCが無効の場合、autorelease済みのオブジェクトが返される  

```objectivec
 +(id) array  
 {  
 	return [[[NSMutableArray alloc] init] autorelease];  
 }  
```

* 例

```objectivec
	{
		// arrayメソッド内でNSMutableArrayオブジェクトが生成され、返される
		id __strong obj = [NSMutableArray array];
	}
```	

```objectivec
	// コンパイラによる擬似コード
	id obj = objc_msgSend(NSMutableArray, @selector(array));
    objc_retainAutoreleasedReturnValue(obj);
    objc_release(obj);
```

1. NSMutableArrayのarrayメソッドを呼び出し
2. retainしてオブジェクトを所有
3. スコープを抜けるのでrelease呼び出し
  
* もうちょっと深く (arrayメソッドを見てみる)

```objectivec
	+ (id) array
	{
		// ARC有効。
		return [[NSMutableArray alloc] init];
	}
```

```objectivec
	+ (id) array
	{
		id obj = objc_msgSend(NSMutableArray, @selector(alloc));
		objc_msgSend(obj, @selector(init));
		return objc_autoreleaseReturnValue(obj);
	}
```

1. NSMutableArrayのalloc, initメソッド呼び出し
2. autorelease呼び出し(autorelease poolに登録)

* 最適化について
 - 一部の環境では、この一連の処理で最適化がされている。
 - objc\_autoreleaseReturnValue()を使用している関数、またはその呼び出し元でobjc_retainAutoreleasedReturnValue()を呼び出している場合、autorelease poolに登録しない
 - iOSではこの最適化は無効らしい
 - ループ内でarray()のようなメソッドを呼ぶときは、ループ全体を@autoreleasepoolで囲うのが無難(らしい)


## 1.4.2 __weak修飾子

### __weak修飾子が提供する機能

* __weak修飾子付き変数は、参照先オブジェクトが破棄されると、nilが代入される
* __weak修飾子付き変数を使うと、autoreleasepoolに登録されたオブジェクトを使うことになる

### 実装をみてみる
```objectivec
    {
        id __weak obj1 = obj;
    }
```

```objectivec
    /* コンパイラによる擬似コード */
    id obj1;
    obj1 = 0;
    objc_storeWeak(&obj1, obj);
    objc_storeWeak(&obj1, 0);
```
* storeWeak()は、第二引数である、代入するオブジェクトのアドレスをキーにして、第一引数の__weak変数のアドレスをweakテーブルに登録する
* 第二引数が0の場合、__weak変数のアドレスをweakテーブルから削除する

### オブジェクトが開放された時の処理
1. objc_release
2. dealloc
3. ... // いろいろな処理
4. objc\_clear\_deallocating

objc\_clear\_deallocatingの処理:
1. weakテーブルから、破棄されるオブジェクトのアドレスがキーであるエントリを取得
2. エントリに含まれる、すべての__weak修飾子付き変数のアドレスに対してnilを代入
3. weakテーブルからエントリを削除
4. 参照カウントテーブルから、オブジェクトアドレスがキーであるエントリを削除

### オブジェクトの即時開放
次のコードはコンパイラにより警告される

```objectivec
	{
		id __weak obj = [[NSObject alloc] init];
	}
```

```objectivec
    /* コンパイラによる擬似コード */
    id obj;
    id tmp = objc_msgSend(NSObject, @selector(alloc));
    objc_msgSend(tmp, @selector(init));
    objc_initWeak(&obj, tmp);
    objc_release(tmp);
    objc_destroyWeak(&object);
```
オブジェクトはすぐにreleaseされ、objにはnilが代入される


### __weak修飾子付き変数を使用してみる

```objectivec
    {
        id __weak obj1 = obj;
        NSLog(@"%@", obj1);
    }
```

```objectivec
    /* コンパイラによる擬似コード */
    id obj1;
    objc_initWeak(&obj1, obj);
    id tmp = objc_loadWeakRetained(&obj1);
    objc_autorelease(tmp);
    NSLog(@"%@", tmp);
    objc_destroyWeak(&obj1);
```

1. objc_loadWeakRetainedで__weak修飾子付き変数の参照先オブジェクトを取り出してretain
2. objc_autoreleaseでそのオブジェクトをautoreleasepoolに登録

* __weak修飾子付き変数が参照するオブジェクトはautorelease poolに登録されるので、@autoreleasepoolブロックが終了するまで有効。
* ただし\__weak修飾子付き変数を使用すると、使用しただけautoreleasepoolに登録しにかかるので、いったん__strong修飾子付き変数に代入してから使用したほうがよい

```objectivec
	id __weak weak_o = obj;
	NSLog(@"%@", weak_o);  // autoreleasepoolに登録される
	NSLog(@"%@", weak_o);  // autoreleasepoolに登録される
	id strong_o = weak_o;
	NSLog(@"%@", strong_o);  // autoreleasepoolに登録されない (__strongなので)
```


### __weak修飾子が使用できないケース
* __weak修飾子に非対応なクラスが存在する
 - ex. NSMachPort class
* これらのクラスは、参照カウントを独自に実装している
* なので__weak修飾子は使えない
* \_\_weak修飾子非対応クラスはクラス宣言で \_\_attribute\_\_((objc\_arc\_weak\_reference\_unavailable)) という属性が追加されている
* 数は少ないしコンパイルエラーになるのであまり気にする必要なし



## 1.4.3 __autoreleasing修飾子
* __autoreleasing修飾子付きの変数にオブジェクトを代入する、というのは、ARCが無効な場合でオブジェクトのautoreleaseを呼ぶ、と等価
* 次のコードで確認してみる:

```objectivec
    @autoreleasepool {
        id __autoreleasing obj = [[NSObject alloc] init];
    }
```

```objectivec
    /* コンパイラによる擬似コード */
    id pool = objc_autoreleasePoolPush();
    id obj = objc_msgSend(NSObject, @selector(alloc));
    objc_msgSend(obj, @selector(init));
    objc_autorelease(obj);
    objc_autoreleasePoolPop(pool);
```

* ARCなしの時のautoreleaseと動作が同じことがわかる



## 1.4.4 参照カウント
* 参照カウントを取得する関数  
``` uintptr_t _objc_rootRetainCount(id obj) ```

* 例をいくつか

```objectivec
    {
        id __strong obj = [[NSObject alloc] init];
        NSLog(@"retain count = %d", _objc_rootRetainCount(obj));
    }
```
```
	retain count = 1
```

```objectivec
    @autoreleasepool {
        id __strong obj = [[NSObject alloc] init];
        id __autoreleasing o = obj;
        NSLog(@"retain count = %d", _objc_rootRetainCount(obj));
    }
```
```
	retain count = 2
```

```objectivec
    {
        id __strong obj = [[NSObject alloc] init];
        @autoreleasepool {
            id __autoreleasing o = obj;
            NSLog(@"retain count = %d in @autoreleasepool", _objc_rootRetainCount(obj));
        }
        NSLog(@"retain count = %d", _objc_rootRetainCount(obj));
    }
```
```
	retain count = 2 in @autoreleasepool
	retain count = 1
```

```objectivec
    @autoreleasepool {
        id __strong obj = [[NSObject alloc] init];
        _objc_autoreleasePoolPrint();
        id __weak o = obj;
        NSLog(@"before using __weak: retain count = %d", _objc_rootRetainCount(obj));
        NSLog(@"class = %@", [o class]);
        NSLog(@"after using __weak: retain count = %d", _objc_rootRetainCount(obj));
        _objc_autoreleasePoolPrint();
    }
```
```
    objc[14481]: ##############
    objc[14481]: AUTORELEASE POOLS for thread 0xad0892c0
    objc[14481]: 1 releases pending.
    objc[14481]: [0x6a85000]  ................  PAGE  (hot) (cold)
    objc[14481]: [0x6a85028]  ################  POOL 0x6a85028
    objc[14481]: ##############
    before using __weak: retain count = 1
    class = NSObject
    after using __weak: retain count = 2
    objc[14481]: ##############
    objc[14481]: AUTORELEASE POOLS for thread 0xad0892c0
    objc[14481]: 2 releases pending.
    objc[14481]: [0x6a85000]  ................  PAGE  (hot) (cold)
    objc[14481]: [0x6a85028]  ################  POOL 0x6a85028
    objc[14481]: [0x6a8502c]         0x6719e40  NSObject
    objc[14481]: ##############
```

* \_objc_rootRetainCountは常に信頼できる数値を返すとは限らない
 - 解放済みのオブジェクトやオブジェクトとして不正なアドレスでも「1」を返す場合がある
 - 複数スレッドから参照されているオブジェクトの参照カウントは、レースコンディション問題のため誤った値が返される可能性がある
 