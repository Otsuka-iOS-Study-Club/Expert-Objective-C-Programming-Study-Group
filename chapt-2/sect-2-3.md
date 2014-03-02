2.3 Blocks の実装
=========================

## 2.3.1 Blockの実態
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
