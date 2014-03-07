2.3 Blocks の実装
=========================

## <a name="section2.3.1">2.3.1 Blockの実態
- clangの-rewrite-objcを使い、Block構文を含んだObjective-CをC++に変換して実装を見てみる
- 下記のBlock構文をC++に変換する
```
int main()
{
	void (^blk)(void) = ^{printf("BlockYn");};
	blk();
	return 0;
}
```
変換後：
```
struct __block_impl {
	void *isa;
	int Flags;
	int Reserved;
	void *FuncPtr;
};
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
	__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
		impl.isa = &_NSConcreteStackBlock;
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
	}
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	printf("BlockYn");
}

static struct __main_block_desc_0 {
	unsigned long reserved;
	unsigned long Block_size;
	} __main_block_desc_0_DATA = {
		0,
		sizeof(struct __main_block_impl_0)
};

int main()
{
	void (*blk)(void) =
	(void (*)(void))&__main_block_impl_0(
		(void *)__main_block_func_0, &__main_block_desc_0_DATA);
	((void (*)(struct __block_impl *))(
		(struct __block_impl *)blk)->FuncPtr)((struct __block_impl *)blk);
	return 0;
}
```

- Blockの無名関数はC言語の関数として変換されている
	- __cselfはthisまたはselfに相当する
```
^{printf("BlockYn")};

//変換後

static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	printf("BlockYn");
}
```
- しかし、変換後のコードではこの__cselfは使われていない
- __cselfの宣言は以下の通り
```
struct __main_block_impl_0 *__cself
```
- ここの__cselfは__main_block_impl_0の構造体へのポインタ
- 構造体は以下の通り
```
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
}
```
- __block_implの構造体
	- フラグと今後のバージョンアップに備えた領域が存在する
```
struct __block_impl {
	void *isa;
	int Flags;
	int Reserved;
	void *FuncPtr;
};
```
- __main_block_desc_0構造体
	- 予約領域とBlockのサイズを持っている

```
struct __main_block_desc_0 {
	unsigned long reserved;
	unsigned long Block_size;
};
```
- これらを初期化する__main_block_impl_0のコンストラクタは以下の通り
```
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
	impl.isa = &_NSConcreteStackBlock;
	impl.Flags = flags;
	impl.FuncPtr = fp;
	Desc = desc;
}
```
- これの呼び出し元は以下の通り
	- 見やすさの為にキャストを取り払う
	- blkに__main_block_impl_0を代入している
```
struct __main_block_impl_0 tmp =
	__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);

struct __main_block_impl_0 *blk = &tmp;
```
- これに対応するBlockのコードは次の通り
	- BlockをBlock型変数のblkに代入を行っている
		- 変換後のソースコードでblkに__main_block_impl_0を代入しているのと等しい
```
void (^blk)(void) = ^{printf("BlockYn");};
```
- この__main_block_impl_0の引数は次のとおり
	- 一つ目の引数はBlockが変換されたCの関数に対するポインタ
	- 二つ目の引数はグローバル変数として初期化された__main_block_desc_0の構造体へのポインタ
```
__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);
```
- 二つ目の引数の初期化部分
```
static struct __main_block_desc_0 __main_block_desc_0_DATA = {
	0,
	sizeof(struct __main_block_impl_0)
};
```
- __main_block_impl_0のサイズを使って初期化されている
- __block_implの構造体を展開したら、次のようになる
```
struct __main_block_impl_0 {
	void *isa;
	int Flags;
	int Reserved;
	void *FuncPtr;
	struct __main_block_desc_0* Desc;
}
```
- この構造体は次のように初期化される
```
isa = &_NSConcreteStackBlock;
Flags = 0;
Reserved = 0;
FuncPtr = __main_block_func_0;
Desc = &__main_block_desc_0_DATA;
```
- FuncPtrとして__main_block_func_0;が代入されている
	- 関数ポインタを使った呼び出し
```
blk();

//次のように変換されている

(*blk->impl.FuncPtr)(blk);
```
- _NSConcreteStackBlockの説明
	- BlockはObjective-Cのオブジェクトでもある
```
struct __main_block_impl_0 {
	void *isa;
	int Flags;
	int Reserved;
	void *FuncPtr;
	struct __main_block_desc_0* Desc;
}
```
- このisaは&_NSConcreteStackBlockで初期化が行われている。
- C言語のclass_t構造体のインスタンスに相当する
- 結論として、BlockはObjective-Cのオブジェクトでもある

## 2.3.2 自動変数値のキャプチャ
- 自動変数値のキャプチャがどのように実装されているのかを解説する
```
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
	const char *fmt;
	int val;
	__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,
			const char *_fmt, int _val, int flags=0) : fmt(_fmt), val(_val) {
		impl.isa = &_NSConcreteStackBlock;
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
	}
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	const char *fmt = __cself->fmt;
	int val = __cself->val;
	printf(fmt, val);
}
static struct __main_block_desc_0 {
	unsigned long reserved;
	unsigned long Block_size;
	} __main_block_desc_0_DATA = {
		0,
		sizeof(struct __main_block_impl_0)
	};
int main()
{
	int dmy = 256;
	int val = 10;
	const char *fmt = "val = %dYn";
	void (*blk)(void) = &__main_block_impl_0(
		__main_block_func_0, &__main_block_desc_0_DATA, fmt, val);
	return 0;
}
```
- Blockで使ってる自動変数が__main_block_impl_0のメンバ変数として追加されている
```
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
	const char *fmt;
	int val;
};
```
- 変数dmyはBlock構文の式で使われていない為追加されていない
- この構造体のインスタンスを初期化しているコンストラクタは次の通り
```
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,
	const char *_fmt, int _val, int flags=0) : fmt(_fmt), val(_val) {
```
- 初期化時に追加されたメンバ変数を引数で初期化している
	- この呼び出し元で引数を確認してみる
```
void (*blk)(void) = &__main_block_impl_0(
	__main_block_func_0, &__main_block_desc_0_DATA, fmt, val);
```
- fmtとvalを使って初期化が行われている
- __main_block_impl_0は次のように初期化されている
```
impl.isa = &_NSConcreteStackBlock;
impl.Flags = 0;
impl.FuncPtr = __main_block_func_0;
Desc = &__main_block_desc_0_DATA;
fmt = "val = %dYn";
val = 10;
```
- Block構文は次の用に変換されている
```
/*変換前
^{printf(fmt, val);}
*/

static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	const char *fmt = __cself->fmt;
	int val = __cself->val;
	
	printf(fmt, val);
}
```
- Block構文で使用されている値が実行されるタイミングでBlockに保存される
- 値を渡しているため、C配列は保存できない

## 2.3.3 __block指定子
- キャプチャした自動変数は書き戻されてはいない
- Block内から値を書き換えることができる変数
	- 静的変数
	- 静的大域変数
	- 大域変数
- 静的変数は変数スコープ外にあるため、アクセスができないはず
```
int global_val = 1;
static int static_global_val = 2;
int main()
{
	static int static_val = 3;
	void (^blk)(void) = ^{
		global_val *= 1;
		static_global_val *= 2;
		static_val *= 3;
	};
	return 0;
}
```
- static_val, static_global_val, global_valを書き換えるBlock
```
int global_val = 1;
static int static_global_val = 2;
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
	int *static_val;
	__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,
			int *_static_val, int flags=0) : static_val(_static_val) {
		impl.isa = &_NSConcreteStackBlock;
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
	}
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
	int *static_val = __cself->static_val;
	global_val *= 1;
	static_global_val *= 2;
	(*static_val) *= 3;
}
static struct __main_block_desc_0 {
	unsigned long reserved;
	unsigned long Block_size;
	} __main_block_desc_0_DATA = {
		0,
		sizeof(struct __main_block_impl_0)
};
int main()
{
	static int static_val = 3;
	blk = &__main_block_impl_0(
		__main_block_func_0, &__main_block_desc_0_DATA, &static_val);
	return 0;
}
```
- 他の変数と違い、static_valはポインタを使ってアクセスを行っている。
```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
	int *static_val = __cself->static_val;
	(*static_val) *= 3;
}
```
- 次は__block指定子を追加したソースを変換してみる
```
/* 変換前
__block int val = 10;
void (^blk)(void) = ^{val = 1;}
*/

struct __Block_byref_val_0 {
	void *__isa;
	__Block_byref_val_0 *__forwarding;
	int __flags;
	int __size;
	int val;
};
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
	__Block_byref_val_0 *val;
	__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,
	__Block_byref_val_0 *_val, int flags=0) : val(_val->__forwarding) {
		impl.isa = &_NSConcreteStackBlock;
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
	}
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	__Block_byref_val_0 *val = __cself->val;
	(val->__forwarding->val) = 1;
}
static void __main_block_copy_0(
struct __main_block_impl_0*dst, struct __main_block_impl_0*src)
{
	_Block_object_assign(&dst->val, src->val, BLOCK_FIELD_IS_BYREF);
}
static void __main_block_dispose_0(struct __main_block_impl_0*src)
{
	_Block_object_dispose(src->val, BLOCK_FIELD_IS_BYREF);
}
static struct __main_block_desc_0 {
	unsigned long reserved;
	unsigned long Block_size;
	void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
	void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = {
	0,
	sizeof(struct __main_block_impl_0),
	__main_block_copy_0,
	__main_block_dispose_0
};
int main()
{
	__Block_byref_val_0 val = {
		0,
		&val,
		0,
		sizeof(__Block_byref_val_0),
		10
	};
blk = &__main_block_impl_0(
__main_block_func_0, &__main_block_desc_0_DATA, &val, 0x22000000);
	return 0;
}
```
- __block変数valは構造体のインスタンスに変換されている
```
__Block_byref_val_0 val = {
	0,
	&val,
	0,
	sizeof(__Block_byref_val_0),
	10
};
```
- 上記の宣言は次の通り
```
struct __Block_byref_val_0 {
	void *__isa;
	__Block_byref_val_0 *__forwarding;
	int __flags;
	int __size;
	int val;
};
```
- valへの代入部分は次のように変換されている
```
/*変換前
^{val = 1;}
*/

static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	__Block_byref_val_0 *val = __cself->val;
	(val->__forwarding->val) = 1;
}
```
- メンバ変数__fowardingは自分自身に対するポインタを持っている
	- それを経由してvalにアクセスしている
	- 詳しい説明は次の章で
- __Block_byref_val_0が__main_block_impl_0と分離している理由は複数のBlockから__blockを使用するため
```
/*変換前
__block int val = 10;
void (^blk0)(void) = ^{val = 0;};
void (^blk1)(void) = ^{val = 1;};
*/

__Block_byref_val_0 val = {0, &val, 0, sizeof(__Block_byref_val_0), 10};

blk0 = &__main_block_impl_0(
	__main_block_func_0, &__main_block_desc_0_DATA, &val, 0x22000000);
	
blk1 = &__main_block_impl_1(
	__main_block_func_1, &__main_block_desc_1_DATA, &val, 0x22000000);
```
- 同じvalのポインタを使用しているため、複数のBlockから同じ__blockを使うことができる
	- 同様に、一つのBlockから複数の__blockを使うことができる
- static_val, static_global_val, global_valを書き換えるBlock
```
int global_val = 1;
static int static_global_val = 2;
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
	int *static_val;
	__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,
			int *_static_val, int flags=0) : static_val(_static_val) {
		impl.isa = &_NSConcreteStackBlock;
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
	}
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
	int *static_val = __cself->static_val;
	global_val *= 1;
	static_global_val *= 2;
	(*static_val) *= 3;
}
static struct __main_block_desc_0 {
	unsigned long reserved;
	unsigned long Block_size;
	} __main_block_desc_0_DATA = {
		0,
		sizeof(struct __main_block_impl_0)
};
int main()
{
	static int static_val = 3;
	blk = &__main_block_impl_0(
		__main_block_func_0, &__main_block_desc_0_DATA, &static_val);
	return 0;
}
```
- 他の変数と違い、static_valはポインタを使ってアクセスを行っている。
```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
	int *static_val = __cself->static_val;
	(*static_val) *= 3;
}
```
- 次は__block指定子を追加したソースを変換してみる
```
/* 変換前
__block int val = 10;
void (^blk)(void) = ^{val = 1;}
*/

struct __Block_byref_val_0 {
	void *__isa;
	__Block_byref_val_0 *__forwarding;
	int __flags;
	int __size;
	int val;
};
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
	__Block_byref_val_0 *val;
	__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,
	__Block_byref_val_0 *_val, int flags=0) : val(_val->__forwarding) {
		impl.isa = &_NSConcreteStackBlock;
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
	}
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	__Block_byref_val_0 *val = __cself->val;
	(val->__forwarding->val) = 1;
}
static void __main_block_copy_0(
struct __main_block_impl_0*dst, struct __main_block_impl_0*src)
{
	_Block_object_assign(&dst->val, src->val, BLOCK_FIELD_IS_BYREF);
}
static void __main_block_dispose_0(struct __main_block_impl_0*src)
{
	_Block_object_dispose(src->val, BLOCK_FIELD_IS_BYREF);
}
static struct __main_block_desc_0 {
	unsigned long reserved;
	unsigned long Block_size;
	void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
	void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = {
	0,
	sizeof(struct __main_block_impl_0),
	__main_block_copy_0,
	__main_block_dispose_0
};
int main()
{
	__Block_byref_val_0 val = {
		0,
		&val,
		0,
		sizeof(__Block_byref_val_0),
		10
	};
blk = &__main_block_impl_0(
__main_block_func_0, &__main_block_desc_0_DATA, &val, 0x22000000);
	return 0;
}
```
- __block変数valは構造体のインスタンスに変換されている
```
__Block_byref_val_0 val = {
	0,
	&val,
	0,
	sizeof(__Block_byref_val_0),
	10
};
```
- 上記の宣言は次の通り
```
struct __Block_byref_val_0 {
	void *__isa;
	__Block_byref_val_0 *__forwarding;
	int __flags;
	int __size;
	int val;
};
```
- valへの代入部分は次のように変換されている
```
/*変換前
^{val = 1;}
*/

static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
	__Block_byref_val_0 *val = __cself->val;
	(val->__forwarding->val) = 1;
}
```
- メンバ変数__fowardingは自分自身に対するポインタを持っている
	- それを経由してvalにアクセスしている
	- 詳しい説明は次の章で
- __Block_byref_val_0が__main_block_impl_0と分離している理由は複数のBlockから__blockを使用するため
```
/*変換前
__block int val = 10;
void (^blk0)(void) = ^{val = 0;};
void (^blk1)(void) = ^{val = 1;};
*/

__Block_byref_val_0 val = {0, &val, 0, sizeof(__Block_byref_val_0), 10};

blk0 = &__main_block_impl_0(
	__main_block_func_0, &__main_block_desc_0_DATA, &val, 0x22000000);
	
blk1 = &__main_block_impl_1(
	__main_block_func_1, &__main_block_desc_1_DATA, &val, 0x22000000);
```
- 同じvalのポインタを使用しているため、複数のBlockから同じ__blockを使うことができる
	- 同様に、一つのBlockから複数の__blockを使うことができる

## 2.3.4 Blockの記憶域
- これまでの説明からBlockと__block変数は、構造体型の自動変数であり、スタック上に生成されたその構造体のインスタンスである
- また、Blockは_NSConcreateStackBlockクラスというObjective-Cのオブジェクトでもあった
- _NSConcreateStackBlockに似たクラスが存在する
	- _NSConcreteStackBlock : スタック上に配置
	- _NSConcreteGlobalBlock : データ領域(.dataセクション)に配置
	- _NSConcreteMallocBlock : ヒープに配置
- Blockが記述される場所によって使い分けられる
```
void (^blk)(void) = ^{printf("Global BlockYn");};
int main()
{
/* main() program */
}
```
このコードを変換すると[2.3.1 Blockの実態](#section2.3.1)で説明したようなBlockが生成され、Block用構造体メンバ変数isaは次のように初期化される
```
impl.isa = &_NSConcreteGlobalBlock;
```
- 大域変数が記述される場所では、自動変数が使用できないため、キャプチャする自動変数も存在しない
- そのため、Block用構造体インスタンスの内容が実行時の状態に依存しないので、プログラム全体で1つだけインスタンスが存在すれば十分

- Block用構造体インスタンスは自動変数をキャプチャした場合のみ、実行時の状態によってその値が変化する
- 次のコードでは1つのBlock構文を何度も使用しているが、キャプチャされる自動変数の値はfor ループごとに変化する
```
typedef int (^blk_t)(int);
    for (int rate = 0; rate < 10; ++rate) {
    blk_t blk = ^(int count){return rate * count;};
}
```
- 次のソースコードのように自動変数をキャプチャしない場合は、Block 用構造体インスタンスは毎回まったく同じものになる
```
typedef int (^blk_t)(int);
    for (int rate = 0; rate < 10; ++rate) {
    blk_t blk = ^(int count){return count;};
}
```
- つまり、Block 構文を大域変数と同じ場所ではなく関数内で使用したときでも、Block が自動変数をキャプチャしない場合は、Block 用構造体インスタンスをプログラムのデータ領域に配置しても問題ない

- *まとめ*

- 以下のときはBlock は_NSConcreteGlobalBlock クラスのオブジェクトとなる。これ以外のBlock 構文によるBlock は、_NSConcreteStackBlock クラスのオブジェクトとなり、スタック上に配置される
	- 大域変数が記述される場所にBlock 構文がある場合
	- Block 構文の式で、キャプチャすべき自動変数を使用していない場合
- __NSConcreteMallocBlockクラスは？
- 前節最後の疑問を解決するために使われる。
	- Block が変数スコープを越えて存在可能な理由
	- \_\_block 変数用構造体のメンバ変数__forwarding の存在理由
- 大域変数に配置されているBlock であれば、変数スコープ外からでも、ポインタ経由で安全に使
用することが可能。しかし、スタックに配置されているBlock では、そのBlock が所属する変
数スコープが終了すれば、そのBlock も破棄されてしまう。\_\_block 変数もスタックに配置されている
ので、同じように、その\_\_block 変数が所属する変数スコープが終了すれば、その\_\_block 変数も破
棄される。
- これを解決するために、Blocks では、Block と\_\_block 変数をスタックからヒープにコピーする方法を提供してる。スタック上に配置されたBlock を、ヒープ上にコピーすることにより、Block 構文が記述されていた変数スコープが終了しても、ヒープ上のBlock は存在し続けることが可能になる。
- そのヒープ上にコピーされたBlock は、_NSConcreteMallocBlock クラスのオブジェクトであるように、Block 用構造体インスタンスのメンバ変数isa が書き換えられる。

```
impl.isa = &_NSConcreteMallocBlock;
```
- また、\_\_block 変数用構造体のメンバ変数\_\_forwarding の存在理由は、\_\_block 変数がスタックに配置されていても、ヒープに配置されていても、正しく\_\_block 変数にアクセス可能にするため。

- 「2.3.5 　\_\_block 変数の記憶域」で詳しく説明するが、\_\_block 変数がヒープに配置されている状態でも、スタック上の__block 変数をアクセスする場合がある。
- その場合でも、スタック上の構造体インスタンスのメンバ変数\_\_forwarding が、ヒープ上の構造体インスタンスを指していれば、スタック上の\_\_block 変数からでも、ヒープ上の\_\_block 変数からでも、正しくアクセスすることが可能。

- Blocks が提供する、コピーの方法とは?
- 実は、ARC が有効のときは、多くの場合コンパイラが適切に判断し、Block を自動的にスタックからヒープにコピーするコードを生成します。
- 例: スタック上のBlockが破棄されてしまうソースコード
```
- typedef int (^blk_t)(int);
blk_t func(int rate)
{
    return ^(int count){return rate * count;};
}
```
- このソースコードは、ARC対応コンパイラにより次のようなソースコードに変換される。
```
blk_t func(int rate)
{
    blk_t tmp = &__func_block_impl_0(
    __func_block_func_0, &__func_block_desc_0_DATA, rate);
    tmp = objc_retainBlock(tmp);
    return objc_autoreleaseReturnValue(tmp);
}
```
- ARC が有効な状態なので、「blk\_t tmp」は、実は、\_\_strong 修飾子が付いた「blk\_t\_\_strong tmp」と同じです。なお、objc4 ランタイムライブラリのruntime/objc-arr.mm を読むとわかりますが、objc_retainBlock 関数の実態は、_Block_copy 関数。つまり、
```
tmp = _Block_copy(tmp);
return objc_autoreleaseReturnValue(tmp);
```
- コメントを追加すると、
```
/*
* Block型に相当する変数tmp に、
* Block構文によるBlock、
* つまり、スタック上に配置された
* Block用構造体のインスタンスが代入されている。
*/
tmp = _Block_copy(tmp);
/*
* _Block_copy関数で、
* スタック上のBlock がヒープ上にコピーされる。
* コピー後、ヒープ上のアドレスをポインタとして変数tmp に代入。
*/
return objc_autoreleaseReturnValue(tmp);
/*
* ヒープ上にあるBlock をObjective-C のオブジェクトとして
* autoreleasepoolに登録してから、そのオブジェクトを返す。
*/
```
- 関数の戻り値としてBlock を返す場合は、コンパイラが自動的にヒープにコピーしてくれる

- コンパイラが適切に判断できない場合は、手動でcopy インスタンスメソッドを呼び、Block をスタックからヒープにコピーする必要がある
- コンパイラが適切に判断できない場合とは、以下の場合
	- メソッドまたは関数の引数に、Block を渡す場合
	ただし、メソッドまたは関数側で、渡ってきた引数を適切にコピーしている場合は、そのメソッドまたは関数呼び出し前に手動でコピーする必要はありません。次のメソッドや関数は、手動コピー不要です。
	- Cocoa フレームワークのメソッドで、かつ、メソッド名にusingBlock などが含まれている場合
	- Grand Central Dispatch のAPI
- 具体例を挙げると、NSArray クラスのenumerateObjectsUsingBlock インスタンスメソッドや、dispatch_async 関数を使用する場合は、手動コピーは不要
- 逆に、NSArray クラスのinitWithObjects インスタンスメソッドにBlock を渡す場合は、手動コピーが必要

```
- (id) getBlockArray
{
    int val = 10;

    return [[NSArray alloc] initWithObjects:
        ^{NSLog(@"blk0:%d", val);},
        ^{NSLog(@"blk1:%d", val);}, nil];
}
```
- このgetBlockArray メソッドは、スタック上に2 つのBlock を生成し、NSArray クラスのinitWithObjects インスタンスメソッドに渡す。getBlockArray メソッド呼び出し元で、NSArrayオブジェクトから取り出したBlock を実行すると
```
id obj = getBlockArray();
typedef void (^blk_t)(void);
blk_t blk = (blk_t)[obj objectAtIndex:0];
blk();
```
- このソースコードはblk()、つまりBlock 実行時に例外が発生して、アプリケーションが強制終了する。
	- getBlockArray 関数の実行終了時に、スタック上のBlock は破棄されているため
- コンパイラが、コピーが必要であるか否か判断せずに、常にコピーしてしまう戦略も取ることは可能だがBlock をスタックからヒープにコピーするのは、それなりにCPU コストがかかる
- Block がスタックに配置されたままでも使用できる場合、Block をスタックからヒープにコピーするのは、無駄にCPU を使うことになる。そのため、このような場合のみ、プログラマに手動でコピーさせることにしたと考えられる。
- このソースコードは、次のように修正すれば動作する。
```
- (id) getBlockArray
{
    int val = 10;

    return [[NSArray alloc] initWithObjects:
        [^{NSLog(@"blk0:%d", val);} copy],
        [^{NSLog(@"blk1:%d", val);} copy], nil];
}
```
- Block 構文に対して、直接copy メソッドを呼ぶことが可能。Block 型変数に対して、copy メソッド呼ぶことも可能。
```
typedef int (^blk_t)(int);

blk_t blk = ^(int count){return rate * count;};

blk = [blk copy];
```

- 既にヒープに配置されたBlock や、プログラムのデータ領域に配置されたBlock に対して、copy メソッドを呼んだ場合、どうなるか?
	- \_NSConcreteStackBlock : スタックスタックからヒープにコピー
	- \_NSConcreteGlobalBlock: プログラムのデータ領域何も起こらない
	- \_NSConcreteMallocBlock: ヒープ参照カウント加算
- Block がどこに配置されていても、copy メソッドによるコピーで何か悪いことが起こるわけではないので不安な場合は、copy メソッドを呼んでおくとよい。

- 何回もcopy メソッドを呼んでコピーしてしまって平気?
```
blk = [[[[blk copy] copy] copy] copy];
```
- このコードは、次のようなソースコードであると解釈できる。
```
{
    blk_t tmp = [blk copy];
    blk = tmp;
}
{
    blk_t tmp = [blk copy];
    blk = tmp;
}
{
    blk_t tmp = [blk copy];
    blk = tmp;
}
{
    blk_t tmp = [blk copy];
    blk = tmp;
}
```
- コメントを追加すると以下のようになる。
```
{
    /*
    * 変数blk に、スタックに配置されたBlock が
    * 代入されているとする。
    */
    blk_t tmp = [blk copy];
    /*
    * 変数tmp に、ヒープに配置されたBlock が代入され、
    * 強い参照によりBlock が所有される。
    */
    blk = tmp;
    /*
    * 変数blk に、変数tmp のBlock が代入され、
    * 強い参照によりBlock が所有される。
    **
    元々代入されていたBlock は、
    * スタック上に配置されているので
    * この代入により影響を受けない。
    **
    この時点でBlock の所有者は
    * 変数blk と変数tmp。
    */
}  /*
    * 変数スコープ終了により、変数tmp が破棄され、
    * 強い参照が消滅し、所有していたBlock を解放する。
    *
    * 変数blk により所有されている状況なので、
    * Blockは破棄されない。
    */
{
    /*
    * 変数blk に、ヒープに配置されたBlock が
    * 代入されている。強い参照により、
    * Blockを所有している状態。
    */

    blk_t tmp = [blk copy];

    /*
    * 変数tmp に、ヒープに配置されたBlock が代入され、
    * 強い参照によりBlock が所有される。
    */

    blk = tmp;

    /*
    * 変数blk への代入が発生するため、
    * 現在代入されているBlock への強い参照が消滅、
    * Blockが解放される。
    *
    * 変数tmp により所有されている状況なので、
    * Blockは破棄されない。
    *
    * 変数blk に、変数tmp のBlock が代入され、
    * 強い参照によりBlock が所有される。
    *
    * この時点でBlock の所有者は
    * 変数blk と変数tmp。
    */
}  /*
    * 変数スコープ終了により、変数tmp が破棄され、
    * 強い参照が消滅し、所有していたBlock を解放する。
    *
    * 変数blk により所有されている状況なので、
    * Blockは破棄されない。
    */
/*
 * 以下繰り返し
 */
```
- このとおり、ARC が有効であれば、問題はない

## 2.3.5 __block変数の記憶域
- \_\_block変数はどのように扱われるか?
- \_\_block 変数は、\_\_block 変数を使用しているBlock がスタックからヒープにコピーされたときに、影響を受ける。
	- \_\_block変数の配置記憶域がスタック: スタックからヒープにコピーされて、Block から所有される
	- \_\_block変数の配置記憶域がヒープ  : Block から所有される   

- 1 つのBlock から使用される\_\_block 変数の場合
    - そのBlock がスタックからヒープにコピーされるときは、使用されているすべての\_\_block 変数もスタック上に配置されているはず。
    - それらの\_\_block 変数がすべてスタックからヒープにコピーされ、Block は\_\_block 変数を所有することになる。
    - そのBlock がヒープにコピー済みの場合にコピーされても、使用している\_\_block 変数に対して何も影響はない。

- 複数のBlock から使用される__block 変数の場合
    - 最初はすべてのBlock がスタックに配置されているので、__block 変数もスタックに配置される。
    - どれか1 つのBlock がスタックからヒープにコピーされるタイミングで、一緒に__block 変数もスタックからヒープにコピーされ、そのBlock から所有される。
    - 残りのBlock がスタックからヒープにコピーされるタイミングでは、コピーされるBlock から__block 変数が所有され、\_\_block 変数の参照カウントが加算される。

- ヒープに配置されたBlock が破棄されれば、使用している__block 変数も解放される。
	- つまり、この考え方は、Objective-C の参照カウントによるメモリ管理とまったく同じ。

- \_\_block 変数用構造体のメンバ変数__forwarding 存在理由について
	- 「\_\_block 変数がスタックに配置されていても、ヒープに配置されていても、正しく\_\_block 変数にアクセス可能にするため。」
- 前述のとおり、Block のコピーにより、\_\_block 変数もスタックからヒープにコピーされるが、スタック上の\_\_block 変数と、ヒープ上の\_\_block 変数に同時にアクセスされる可能性がある。
```
\_\_block int val = 0;

void (^blk)(void) = [^{++val;} copy]; (1)

++val;

blk();

NSLog(@"%d", val);
```
- \_\_block 変数を使っているBlock 構文を、copy メソッドでコピーしている(1)。
- コピーした時点で、Block と__block 変数が、両方ともにスタックからヒープにコピーされている。
- \_\_block 変数を使っているところを見ると、初期化のあとは、Block 構文の式で使用されている。
```
^{++val;} (2)
```
そして、Block 構文のあとで、Block と関係なく使用されている。
```
++val; (3)
```
これらのソースコードは、両方とも次のソースコードとして変換されている。
```
++(val.__forwarding->val);
```
- Block 構文が変換された関数では、この変数val (2)はコピーされたヒープ上の\_\_block 変数用構造体インスタンスであり、Block と関係なく使用しているほうの変数val (3)はコピーされる前のスタック上の\_\_block 変数用構造体インスタンス。
- ただし、スタック上の\_\_block 変数用構造体インスタンスは、\_\_block 変数がスタックからヒープにコピーされる際に、メンバ変数__forwarding の値を、コピー先であるヒープ上の\_\_block 変数用構造体インスタンスのアドレスに書き換えている。

- この仕組みにより、同じ__block 変数にアクセスすることが可能になっている。

## 2.3.6 オブジェクトのキャプチャ
- 次のソースコードは、NSMutableArray クラスのオブジェクトを生成し所有しますが、代入先の__strong 修飾子付き変数のスコープがすぐに終了するため、オブジェクトはすぐに解放、破棄される。
```
{
    id array = [[NSMutableArray alloc] init];
}
```
- この変数array を使用するBlock 構文を書いてみると以下のようになる。
```
blk_t blk;
{
    id array = [[NSMutableArray alloc] init];
    blk = [^(id obj) { 
        [array addObject:obj];
        NSLog(@"array count = %ld", [array count]);
    } copy];
}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
```
- スコープの終了とともに変数array は破棄され強い参照が消失するので、変数array に代入されたNSMutableArray クラスのオブジェクトは解放、破棄されているはずだが、ソースコードの実行結果は次の通りになる。
```
array count = 1
array count = 2
array count = 3
```
- これは、変数array に代入されたNSMutableArray クラスのオブジェクトが、変数スコープを越えて存在していることを意味する。
- コンパイラによる変換後のソースコードは以下のようになる。
```
/* Block 用構造体/関数部分*/
struct __main_block_impl_0 {
     struct __block_impl impl;
     struct __main_block_desc_0* Desc;
     id __strong array;
     __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,
             id __strong _array, int flags=0) : array(_array) {
         impl.isa = &_NSConcreteStackBlock;
         impl.Flags = flags;
         impl.FuncPtr = fp;
         Desc = desc;
     }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself, id obj) {
    id __strong array = __cself->array;
    [array addObject:obj];
    NSLog(@"array count = %ld", [array count]);
}

static void __main_block_copy_0(struct __main_block_impl_0 *dst,
        struct __main_block_impl_0 *src)
{
    _Block_object_assign(&dst->array, src->array, BLOCK_FIELD_IS_OBJECT);
}

static void __main_block_dispose_0(struct __main_block_impl_0 *src)
{
    _Block_object_dispose(src->array, BLOCK_FIELD_IS_OBJECT);
}

static struct __main_block_desc_0 { (1)
    unsigned long reserved;
    unsigned long Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = {
    0,
    sizeof(struct __main_block_impl_0),
    __main_block_copy_0,
    __main_block_dispose_0
};
```

```
/* Block 構文、Block 使用部分*/
blk_t blk;
{
    id __strong array = [[NSMutableArray alloc] init];

    blk = &__main_block_impl_0(
    __main_block_func_0, &__main_block_desc_0_DATA, array, 0x22000000);
    blk = [blk copy];
}

(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);
(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);
(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);

```
- NSMutableArray クラスのオブジェクトが代入された、キャプチャされる自動変数arrayはBlock 用の構造体に、__strong 修飾子付きのメンバ変数として入っている。

```
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    id __strong array;
};
```

- 「1.3.4 　ルール」で説明したとおり、Objective-C では、C 言語の構造体には\_\_strong 修飾子付きの変数を入れることができません。
- しかし、Objective-C のランタイムライブラリは、Block がスタックからヒープにコピーされるタイミング、およびヒープ上のBlock が破棄されるタイミングを確実に把握できるため、Block 用の構造体に\_\_strong 修飾子または\_\_weak 修飾子付き変数が入っていても、初期化と破棄のために適切に処理することが可能。
- それには、\_\_main\_block\_desc\_0 構造体(1)に増えているメンバ変数copy とdispose、またそのメンバ変数にポインタとして代入される\_\_main\_block\_copy\_0 関数と\_\_main\_block\_dispose\_0 関数を使用する。
- このソースコードにおけるBlock 用構造体には、__strong 修飾子付きのオブジェクト型変数array が入っているので、この変数array に代入されているオブジェクトを適切に管理する必要があります。
- そこで\_\_main\_block\_copy\_0 関数は、\_Block\_object\_assign 関数を使用して、対象のオブジェクトをBlock 用構造体のメンバ変数array に代入し所有します。
```
static void __main_block_copy_0(struct __main_block_impl_0 *dst,
        struct __main_block_impl_0 *src)
{
    _Block_object_assign(&dst->array, src->array, BLOCK_FIELD_IS_OBJECT);
}
```
- \_Block\_object\_assign 関数は、対象の構造体メンバ変数にオブジェクトを代入し、retain インスタンスメソッドに相当する関数を呼び出す。
- また、\_\_main\_block\_dispose\_0 関数は、\_Block\_object\_dispose 関数を使用して、Block 用構造体のメンバ変数array に代入されているオブジェクトを解放する。
```
static void __main_block_dispose_0(struct __main_block_impl_0 *src)
{
    _Block_object_dispose(src->array, BLOCK_FIELD_IS_OBJECT);
}
```

- \_Block\_object\_dispose 関数は、対象の構造体メンバ変数に代入されているオブジェクトの、release インスタンスメソッドに相当する関数を呼び出す。

- \_\_main\_block\_copy\_0 関数（以降copy 関数）と\_\_main\_block\_dispose\_0 関数（以降dispose 関数）、\_\_main\_block\_desc\_0 構造体のメンバ変数copy とdispose にポインタが代入されているが、変換後のソースコードでは、これらの関数が、どこから呼び出されるのか?

- これらの関数は、Block がスタックからヒープにコピーされるとき、またはヒープ上のBlock が破棄されるときに呼び出される。
    - copy :  関数スタック上のBlock がヒープにコピーされるとき
        - Block のcopy インスタンスメソッドを呼んだとき

        - 関数の戻り値としてBlock を返したとき
            - コンパイラが自動的に、copyを呼び出すのと同義の_Block_copy 関数を対象のBlock を引数に呼び出す。
        - クラスの__strong 修飾子付きid 型もしくはBlock 型メンバ変数にBlock を代入したとき
            - コンパイラが自動的に、copyを呼び出すのと同義の_Block_copy 関数を対象のBlock を引数に呼び出す。
        - メソッド名にusingBlock が含まれるCocoa フレームワークのメソッド、またはGrand Central Dispatch のAPI にBlock を渡したとき
            - そのメソッドや関数内部で、渡されたBlock に対してBlock のcopy インスタンスメソッドもしくは_Block_copy 関数を呼び出す。

    - dispose : 関数ヒープ上のBlock が破棄されるとき
        - ヒープにコピーされたBlock が解放され、誰からも所有されなくなり破棄されたタイミングで呼び出されます。オブジェクトのdealloc インスタンスメソッドに相当する。

- ソースコード上ではこれらのタイミングでスタック上のBlock がヒープにコピーされるように見えるが、結局のところ**「\_Block\_copy 関数が呼び出されるとき」**に集約されます。

- この仕組みにより、__strong 修飾子付き自動変数を使うことでBlock にキャプチャされたオブジェクトは、変数スコープを越えて存在できる。

- copy 関数とdispose 関数を使用する仕組みは、\_\_block 変数を使用する際にも使われている。
```
static void __main_block_copy_0(
    struct __main_block_impl_0*dst, struct __main_block_impl_0*src)
{
    _Block_object_assign(&dst->val, src->val, BLOCK_FIELD_IS_BYREF);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src)
{
    _Block_object_dispose(src->val, BLOCK_FIELD_IS_BYREF);
}
```
- 変換後のソースコードは、Block 用構造体に関する部分はほぼ同じで、違いは次の通り。
    - オブジェクト: BLOCK_FIELD_IS_OBJECT
    - \_\_block 変数BLOCK_FIELD_IS_BYREF
- BLOCK_FIELD_IS_OBJECT とBLOCK_FIELD_IS_BYREFにより、copy 関数とdispose 関数の対象が、オブジェクトなのか__block 変数なのかを区別している。

- しかし、copy 関数でキャプチャしたオブジェクトを所有し、dispose 関数でキャプチャしたオブジェクトを解放したのとまったく同じで、copy 関数で使用している\_\_block 変数を所有し、dispose関数で使用している\_\_block 変数を解放する。

- なお、先ほどのソースコードで、もしBlock のcopy インスタンスメソッドを呼んでいない場合はプログラムの強制終了となる。
    - キャプチャした__strong 修飾子付きのオブジェクト型自動変数値の所有は、\_Block\_copy 関数が呼び出されるときに限られるので、オブジェクトをキャプチャしていても変数スコープとともに破棄されてしまう。

```
blk_t blk;
{
    id array = [[NSMutableArray alloc] init];
    blk = ^(id obj) {
        [array addObject:obj];
        NSLog(@"array count = %ld", [array count]);
    };
}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);

```

- このため、Block からオブジェクト型自動変数を使用する場合は、次の場合以外、Block のcopyインスタンスメソッドを呼ぶことを推奨する。
    - 関数の戻り値としてBlock を返すとき
    - クラスの__strong 修飾子付きid 型もしくはBlock 型メンバ変数にBlock を代入するとき
    - メソッド名にusingBlock が含まれるCocoa フレームワークのメソッド、またはGrand Central Dispatch のAPI にBlock を渡すとき

## 2.3.7 __block変数とオブジェクト
\_\_block 指定子は、自動変数であれば、どんな型にも指定することが可能。Objective-C のオブジェクトを代入するための、id 型自動変数に指定すると以下のようになる。
```
__block id obj = [[NSObject alloc] init];
```
実は、このソースコードは次のソースコードと等価。
```
__block id __strong obj = [[NSObject alloc] init];
```
- このソースコードをclang で変換すると、次のようなソースコードとなる。
```
/* __block 変数用構造体部分*/
struct __Block_byref_obj_0 {
    void *__isa;
    __Block_byref_obj_0 *__forwarding;
    int __flags;
    int __size;
    void (*__Block_byref_id_object_copy)(void*, void*);
    void (*__Block_byref_id_object_dispose)(void*);
    __strong id obj;
};

static void __Block_byref_id_object_copy_131(void *dst, void *src) {
    _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}

static void __Block_byref_id_object_dispose_131(void *src) {
    _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
```

```
/* __block 変数宣言部分*/
__Block_byref_obj_0 obj = {
    0,
    &obj,
    0x2000000,
    sizeof(__Block_byref_obj_0),
    __Block_byref_id_object_copy_131,
    __Block_byref_id_object_dispose_131,
    [[NSObject alloc] init]
};
```
- \_\_block 変数が、\_\_strong 修飾子付きの、id 型またはオブジェクト型自動変数である場合、Blockと同様なことが行われる。
    - \_\_block 変数がスタックからヒープにコピーされるときは、\_Block\_object\_assign 関数が使用され、\_\_block 変数に代入されたオブジェクトが所有される。
    - ヒープ上の\_\_block 変数が破棄されるときは、\_Block\_object\_dispose 関数が使用され、\_\_block変数に代入されたオブジェクトが解放される。

- \_\_weak 修飾子ではどうなるのか?
- Block で\_\_weak 修飾子付きid 型変数を使用する場合
```
blk\_t blk;
{
    id array = [[NSMutableArray alloc] init];
    id __weak array2 = array;
    blk = [^(id obj) {
        [array2 addObject:obj];
        NSLog(@"array2 count = %ld", [array2 count]);
    } copy];
}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);

```

このソースコードの実行結果は、「2.3.6 　オブジェクトのキャプチャ」の結果と異なります。
```
array2 count = 0
array2 count = 0
array2 count = 0
```
これは、\_\_strong 修飾子付き変数array が、その変数スコープを終了すると同時に解放、破棄され、\_\_weak 修飾子付き変数array2 にnil が代入されたためである。

\_\_block 指定子と__weak 修飾子を同時に指定する場合
```
blk_t blk;
{
    id array = [[NSMutableArray alloc] init];
    __block id __weak array2 = array;

    blk = [^(id obj) {
        [array2 addObject:obj];
        NSLog(@"array2 count = %ld", [array2 count]);
    } copy];
}

blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);

```
このソースコードの実行結果も、先ほどと同じ。

```
array2 count = 0
array2 count = 0
array2 count = 0
```
\_\_block 指定子が付いていても、\_\_strong 修飾子付き変数array が、その変数スコープを終了すると同時に解放、破棄され、\_\_weak 修飾子付き変数array2 にnil が代入されるため。

なお、\_\_unsafe\_unretained 修飾子付き変数は単なるポインタと同じなので、Block から使用する場合も、\_\_block 変数に付加する場合も、\_\_strong 修飾子や\_\_weak 修飾子のような処理は行われない。
\_\_unsafe\_unretained 修飾子付変数を使用する際は、dangling pointer による、破棄されたはずのオブジェクトへのアクセスに注意する必要がある。

\_\_autoreleasing 修飾子は、Block との同時使用は想定された使用方法ではないため、そもそも使用する必要はありません。\_\_block 指定子と同時に使用した場合、コンパイルエラーとなる。
```
__block id __autoreleasing obj = [[NSObject alloc] init];
```
変数obj は、\_\_autoreleasing 修飾子と__block 指定子を同時に指定しているが、これは次のようなコンパイルエラーとなります。
```
error: __block variables cannot have __autoreleasing ownership
__block id __autoreleasing obj = [[NSObject alloc] init];
```
## 2.3.8 Blockによる循環参照

## 2.3.9 copy/release
