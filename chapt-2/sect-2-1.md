2.1 Blocks概要
========================================
## 2.1.1 Blocksとは何か?
- C言語に対する機能拡張
- 一言で表すと「自動変数の値を伴った無名関数」
- 他言語で言うAnonymous Function, Lambda, Closure
- 名前のない関数がソースコードに存在でき、呼び出すことができる
- 自動変数（ローカル変数）の値を保持できるため、クラスをわざわざ宣言する必要がない

Blockを使わない例
```
@interface ButtonCallbackObject : NSObject
{
	int buttonId_;
}
@end
@implementation ButtonCallbackObject
- (id) initWithButtonId:(int)buttonId
{
	self = [super init];
	buttonId_ = buttonId;
	return self;
}
- (void) callback:(int)event
{
	NSLog(@"buttonId:%d event=%dYn", buttonId_, event);
}
@end

void setButtonCallbacks()
{
	for (int i = 0; i < BUTTON_MAX; ++i) {
		ButtonCallbackObject *callbackObj =
			[[ButtonCallbackObject alloc] initWithButtonId:i];
		setButtonCallbackUsingObject(BUTTON_IDOFFSET, callbackObj);
	}
}
```

Blockを使った場合、記述量を減らせる。
```
void setButtonCallbacks()
{
	for (int i = 0; i < BUTTON_MAX; ++i) {
		setButtonCallbackUsingBlock(BUTTON_IDOFFSET + i, ^(int event) {
		printf("buttonId:%d event=%dYn", i, event);
		});
	}
}
```
