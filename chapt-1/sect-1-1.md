
1.1 Automatic Reference Counting とは何か? 
========================================
## ARC (Automatic Reference Counting) とは？
- 直訳すると自動参照カウント。
- 参照カウントによるメモリ管理の自動化。
- iOS 5 以降。(Xcode 4.2 以降)
- ARC により、参照カウンタを保持/解放する retain/release を書かなくてよくなる。
- Apple LLVM コンパイラのコンパイラオプションにより ARC が有効にされる。
	- コンパイル時に解決される。(ランタイムではない)
