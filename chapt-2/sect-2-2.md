2.2 Blocksの仕様
=========================

## 2.2.1 Block 構文
- Block Literal Syntaxの仕様
	- 関数名が存在しない
	- 「^」が付く
	- ^ 戻り値の型 (引数リスト) {式}
		- 戻り値の型と引数リストは省略可能

省略していないBlock構文の例
```
^void (int event) {
	printf("buttonId:%d event=%dYn", i, event);
}
```

省略した場合
```
^(int event) {
	printf("buttonId:%d event=%dYn", i, event);
}
```

## 2.2.2 Block 型変数
BlockはCの関数定義と同様にポインタ型変数に代入できる
```
int (^blk)(int) = ^(int count){return count + 1;};
```

同様にBlock型変数からBlock型変数の代入も可能
```
int (^blk1)(int) = blk;

int (^blk2)(int);
blk2 = blk1;
```

関数のパラメータとしても指定可能
尚、typedefを使うことでシンプルにできる
```
typedef int (^blk_t)(int);

void func(int (^blk)(int))
{

void func(blk_t blk)
{
```

戻り値としても指定可能
```
typedef int (^blk_t)(int);

int (^func()(int))
{
return ^(int count){return count + 1;};
}

blk_t func()
{
```

Block型変数の呼び出し方法は以下の通り
```
int result = blk(10);

int func(blk_t blk, int rate)
{
	return blk(rate);
}

//Objective Cの場合
typedef int (^blk_t)(int);
blk_t blk = ^(int count){return count + 1;};
blk_t *blkptr = &blk;
(*blkptr)(10);Blockの実態
```

## 2.2.3 自動変数値のキャプチャ
- Blockでは「自動変数の値を伴った」ことを「自動変数の値をキャプチャする」と表現する。
```
int main()
{
	int dmy = 256;
	int val = 10;
	const char *fmt = "val = %dYn";
	void (^blk)(void) = ^{printf(fmt, val);};
	val = 2;
	fmt = "These values were changed. val = %dYn";
	blk();
	return 0;
}
```
- 上記の実行結果はval = 2ではなくval = 10となる。
- blk()の実行時点の変数が保存される。なので、書き換え後のval = 10ではなくval = 2が結果として表示される。
- キャプチャとはこのこと。

## 2.2.4 __block 指定子
- Block内でキャプチャされた変数は書き換えができない
- 書き換えを行いたい場合は、__block指定子を付ける必要がある。
	- この様な変数を「__block変数と呼ぶ」
```
__block int val = 0;
void (^blk)(void) = ^{val = 1;};
blk();
printf("val = %dYn", val);
```

## 2.2.5 キャプチャした自動変数
- Blockでキャプチャした__block変数ではない変数に値を代入しようとするとコンパイルエラーになる。
- Objective-Cのオブジェクトをキャプチャして、そのオブジェクトを変更した場合はコンパイルエラーにはならない。
```
id array = [[NSMutableArray alloc] init];
void (^blk)(void) = ^{
	id obj = [[NSObject alloc] init];
	[array addObject:obj];
};
```
キャプチャしたNSMutableArrayのポインタarrayに代入していないため上記の操作はコンパイルエラーにならない。
逆に下はエラーとなる
```
id array = [[NSMutableArray alloc] init];
	void (^blk)(void) = ^{
	array = [[NSMutableArray alloc] init];
};

error: variable is not assignable (missing __block type specifier)
	array = [[NSMutableArray alloc] init];
	~~~~~ ^
```
もし上記の操作を行いたい場合は__block指定子を付ける必要がある
```
__block id array = [[NSMutableArray alloc] init];

void (^blk)(void) = ^{
	array = [[NSMutableArray alloc] init];
};
```

- C言語の配列はBlockではキャプチャできない。明示的にポインタを使用する必要がある。
```
const char *text = "hello";
void (^blk)(void) = ^{
	printf("%cYn", text[2]);
};
```
