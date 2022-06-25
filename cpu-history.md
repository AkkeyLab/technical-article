# Apple Silicon から学ぶ CPU の歴史

## Introduction
皆さんは現在利用しているコンピュータの CPU アーキテクチャをご存知でしょうか？アーキテクチャと言っても MVVM, Clean Architecture のようなソフトウェアアーキテクチャではなく、命令セットアーキテクチャと呼ばれるものになります。  
例えば、Intel であれば x86_64, Apple Silicon であれば arm64 が命令セットアーキテクチャです。

本稿では Apple Silicon を中心に CPU に関する歴史を文系・理系関係なく楽しんでいただけるように執筆しました。  
日頃使っている開発環境がどのようなハードウェア設計のもとで動作しているかを知ることで、Apple の素晴らしいハードウェアが更に好きになること間違いなしです！

## 前提
本稿では基礎知識の無い方にも読んでいただけるように心がけて執筆いたしました。そのため、例外を省略して説明している箇所があります。  
例えば「Intel は x86_64 を採用」と言っても、実際には x86_64 と互換性のある様々な製品をリリースしていますし、過去には IA-64 など異なる命令セットアーキテクチャも採用していました。

しかし、内容の間違いや誤解を招く表現が含まれてしまっているかもしれません。本稿は GitHub で管理されており、最新の状態が閲覧できるだけでなく、皆様からのご指摘にも対応可能となっております。ぜひ、ご活用ください。  
GitHub: https://github.com/AkkeyLab/technical-article

## 1. x86_64 / arm64
### 結論
Intel は x86_64 を採用しており、CISC という設計手法がベースになっています。これに対して、Apple Silicon は arm64 を採用しており、RISC という設計手法がベースになっています。

|  CPU  |  命令セットアーキテクチャ  |  設計手法  |
| ---- | ---- | ---- |
|  Intel  |  x86_64  |  CISC  |
|  Apple Silicon  |  arm64  |  RISC  |

### Instruction Set Architecture
まず、我々が見聞きすることの多い CPU メーカとして Intel, AMD があります。これらのメーカが販売する製品の多くで Windows シリーズが動作します。なぜ、CPU メーカが異なるにも関わらず、同様の OS が動作するのでしょうか。その鍵を握るのが命令セットアーキテクチャ（Instruction Set Architecture）です。

CPU は命令セットアーキテクチャという設計の基礎に基づいて開発されています。イメージとしては、建築方法としての木造と鉄筋のようなものです。  
この命令セットアーキテクチャとして Intel, AMD は共に x86_64 もしくは互換のあるものを採用しているため、同様の OS が起動可能なのです。誤解を恐れず言うのであれば、もしあなたが x86_64 互換のある CPU を自作すれば、Windows も起動可能ということです。

### CISC / RISC
では、x86_64, arm64 にはどのような違いがあるのでしょうか。  
この2つは命令方法（計算方法）に大きな違いがあり、x86_64 は CISC (Complex Instruction Set Computer), arm64 は RISC (Reduce Instruction Set Computer) という設計手法を採用しています。CISC は少ない命令回数で処理を実行するため、1命令あたりの処理が複雑です。逆に RISC は1命令あたりの処理が単純であるため、命令回数は多くなります。

学生時代、数学で公式を頑張って覚えた経験はありますでしょうか。この公式を利用すると答えを導き出すまでのステップを大幅に短縮できることが多いです。これが OS から見た CISC の特徴です。  
ただ、公式の証明が難しいことと同じように、複雑な命令（公式）を解読するための CPU 回路は複雑で、多くのエネルギーを必要とします。  
それであれば、公式を使うのをやめて足し算などの単純な計算のみで計算しよう！というのが RISC の特徴です。足し算であれば殆どの人が計算できることと同じように、CPU 回路も単純で省エネです。  

いかがでしょうか、Apple Silicon が低消費電力を実現できた理由などが見えてきたのではないでしょうか。

## 2. Rosetta / Rosetta2
Rosetta2 とは Intel CPU 用のアプリケーションを Apple Silicon 上で実行させるためのシステムです。  
実は、Apple の主要コンピュータに搭載する CPU の変更は今回が初めてではなく、3回目なのです。初代 MC68000 から PowerPC という RISC タイプ CPU へ、2回目は PowerPC から Intel への変更でした。2回目のタイミングで使用されたのが Rosetta だったのです。  
よって、Apple Silicon 上で動作する Rosetta には2世代目の意味でナンバリングがされることになりました。

### Rosetta の進化   
命令セットアーキテクチャの説明で書いたように、RISC と CISC の違いは命令方法でした。ですから、PowerPC 用のアプリケーションは Intel と直接命令のやり取りが出来ないのです。ここで翻訳作業をしているのが Rosetta なのです。  
そんな Rosetta はパフォーマンスが非常に悪かったと言われています。しかし、これは不思議なことではありません。なぜなら、実行する全ての命令に対して翻訳してあげなければならないからです。

しかし、Rosetta2 では多くの改善が施され、パフォーマンスの低下の少ないアプリケーション実行が可能となりました。これはシステム的進化だけでなく Apple Silicon に施された設計の工夫も大きく貢献していると言われています。  

Rosetta2 や Apple Silicon に関する内部情報の殆どが非公開なため、ソフトウェアの挙動からの推測や噂レベルのものが多いです。Rosetta2 に関してはマイナビニュースで公開されている2つの記事（参考文献）が参考になるかもしれません。  
Apple Silicon に関して興味深い記事を発見しました。「How x86 to arm64 Translation Works in Rosetta 2（Rosetta 2のx86からarm64への変換はどのように動作するのか）」というタイトルの記事内で Apple Silicon は　Intel CPU 用アプリケーションの実行を手助けする設計になっていると説明されています。  
> 4/ So Apple simply cheated. They added Intel's memory-ordering to their CPU. When running translated x86 code, they switch the mode of the CPU to conform to Intel's memory ordering.

引用：https://twitter.com/ErrataRob/status/1331735383193903104

メモリアクセス高速化のための Intel CPU 独自の仕組みを Apple Silicon 内にも組み込むことで、Rosetta2 による変換で生まれる実行速度低下を最小限に抑えているようです。

## 3. Change CPU Architecture by Apple
Rosetta について解説する中で登場した Apple が採用してきた歴代の CPU に関してご紹介します。ここまでを理解することで Macintosh についてまとめられた Wikipedia から得られる情報量は格段に増加すること間違いありません。

|  CPU  |  命令セットアーキテクチャ  |  設計手法  |  移行ツール  |
| ---- | ---- | ---- | ---- |
|  MC68000  |  680x0 |  CISC  |  -  |
|  PowerPC  |  Power Architecture  |  RISC  |  Mac 68K Emulator |
|  Intel  |  x86_64  |  CISC  |  Rosetta  |
|  Apple Silicon  |  arm64  |  RISC  |  Rosetta2  |

- 4つの CPU を採用してきており、3回の移行経験がある
- 設計手法は交互に変っているため、移行の難易度が高い
- Apple Silicon は PowerPC と同じ設計手法

Apple Silicon 発表当初、私は「所詮 iPhone に搭載されるプロセッサの進化系でしょ？ Xcode なんて動かないでしょ？」と信用していませんでした。しかし、どのような点で iPhone 搭載 CPU と同じなのかを紐解くうちにその考えが大きく間違っていることに気づいていきました。  
そして、今では Apple Silicon の大ファンです。

---

最後までお読みいただきありがとうございます。  
文系・理系関係なく楽しんでいただけるように執筆したと書きましたが、非常に難易度の高い内容に感じた方もいたかもしれません。今年の iOSDC はオフライン開催が予定されておりますので、ぜひ私を探して質問攻めしてやってください。  
また、本稿は GitHub で管理されておりますので、気軽に issue などの形で質問や修正依頼いただけますと幸いです。

## 4. Bibliographic References
- [自作エミュレータで学ぶx86アーキテクチャ コンピュータが動く仕組みを徹底理解！](https://www.amazon.co.jp/dp/B0148FQNVC)
- [Embedded System  - CISC vs RISC](http://www.sharetechnote.com/html/EmbeddedSystem_CISC_RISC.html)
- [How x86 to arm64 Translation Works in Rosetta 2](https://www.infoq.com/news/2020/11/rosetta-2-translation/)
- [再来する Rosetta を読み解く](https://news.mynavi.jp/article/osxhack-267/)
- [Rosetta2 はこうなっていた!](https://news.mynavi.jp/article/osxhack-273/)
