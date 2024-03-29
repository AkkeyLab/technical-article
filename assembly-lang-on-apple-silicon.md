# Apple Silicon のためのアセンブリ入門

## Introduction
Apple Silicon、皆さんはどれくらい使い倒していますか？
現代の令和においても、我々は CPU と非常に近い距離で対話する手段を持っています。それがアセンブリです。
アセンブリは機械語に非常に近い低水準言語であり、CPU アーキテクチャによって記述方法も異なります。Swift などの高水準言語では記述方法が言語のバージョンアップ以外で変わることは少ないため、この特性に疑問を持つ人もいるかもしれません。

本稿では、アセンブリを通じて Apple Silicon を操作する方法を紹介します。初めは仕事に直接活かせないように思えるかもしれませんが、Apple Silicon と近い距離で対話することは意義深いものです。きっと Apple Silicon がさらに魅力的に感じられるでしょう！

## 前提
本稿では基礎知識の無い方にも読んでいただけるように心がけて執筆いたしました。そのため、例外を省略して説明している箇所があります。  
例えば「Apple Silicon のため」と言っても、ARMアーキテクチャを採用した CPU に共通するもので、他社製品にも通用する事柄だったりします。

しかし、内容の間違いや誤解を招く表現が含まれてしまっているかもしれません。本稿は GitHub で管理されており、最新の状態が閲覧できるだけでなく、皆様からのご指摘にも対応可能となっております。ぜひ、ご活用ください。  
GitHub: https://github.com/AkkeyLab/technical-article

## アセンブリとは
アセンブリ（Assembly）は日本語で「組み立て」を意味する単語で、本稿ではアセンブリ言語のことを指します。アセンブリはプログラミング言語の一つで、機械語と 1:1 に対応した言語のことを言います。表現の一つとして、アセンブラ言語とも呼ばれますが、本稿ではアセンブリ言語と表現し、アセンブリで書かれたプログラムを機械語に変換するツールをアセンブラ（Assembler）と呼ぶことにします。さらに、機械語に変換すことをアセンブル（Assemble）と呼ぶので、アセンブリをアセンブラでアセンブルすると説明することができます。

ところで、皆さんは「ファービー」という玩具をご存知ですか？アメリカの Tiger Electronics が1998年に発売し、日本では株式会社タカラトミーから販売されました。この玩具は、筐体に内蔵された各種センサーからの入力を元に会話や歌を披露してくれるという特徴を持ち、当時多くの注目を集めました。  
実は、この初代ファービー、アセンブリでプログラミングされていたことが分かっています（参考文献：*1）。

## Hello, World!
このセクションでは、アセンブリを使用して `Hello, World!` という文字列を表示するプログラムを作成し、実行する手順を説明します。アセンブリを使用することで、コンピュータの低レベルな動作やシステムコールの利用方法を理解することができます。

1. 利用するシステムコールの把握
2. システムコールの引数に指定する値をレジスタに格納
3. システムコールの実行
4. プログラムの正常終了
5. `as` コマンドを使用してアセンブルし、オブジェクトファイル（拡張子 `.o` ）を生成
6. `ld` コマンドを使用してリンクし、実行可能ファイル（バイナリ）を生成

上記の手順を実践するだけで、アセンブリでのコーディングからプログラムの実行までを行うことができます。  
ここで、システムコールという用語が多く登場していることに気づくかと思います。実は、機械語に近いアセンブリであっても、CPU を自由自在に操作できるわけではなく、カーネルに対してシステムコールという命令を行うことで処理を実行することになります。例えば、標準出力を行う場合には、システムコールを使用してカーネルに出力を依頼します。これは Swift で `print` 関数を用いて標準出力を行う感覚に近いかもしれません。

### 1. 利用するシステムコールの把握
アセンブリで利用するシステムコールを把握するためには、Apple が提供する `syscalls.master` というファイル（参考文献：*2）が参考になります。このファイルには、利用可能なシステムコールの一覧が記載されています。  
今回は `Hello, World!` という文字列を表示したいので、標準入出力が可能な `write` システムコールが利用できそうです。以下に `write` システムコールに関して記述された箇所を `syscalls.master` ファイルから抜粋しました。

```
4	AUE_NULL	ALL	{ user_ssize_t write(int fd, user_addr_t cbuf, user_size_t nbyte); }
```

この抜粋した行には、 `write` システムコールの情報が含まれています。まず、右端の部分にはシステムコールの定義があります。そして、左端の数値はシステムコールナンバーを示しており、アセンブリコード内ではこの番号を利用します。このように、定義から逆引きする形でシステムコールナンバーを特定することになります。  
なぜ、このような逆引きを行う必要があるのでしょうか。実は、アセンブリでシステムコールの単語表現をそのまま使用することができないのです。そのため、システムコールナンバーを調べる必要があるのです。初めはやや難解に感じるかもしれませんが、本稿を読み終える頃には「標準入出力はシステムコール4番」と即答できるようになっていることでしょう。

### 2. システムコールの引数に指定する値をレジスタに格納
利用したいシステムコールが決まったら、その詳細な定義や使用方法を知るために `man` コマンドを利用することができます。具体的には、 `man` コマンドにシステムコール名を指定して実行します。ちなみに、 `man` コマンドの正体は `man man` で調べることができますが、コマンドのドキュメントを表示してくれるコマンドです。

```sh
man write
```

今回の場合、上記のようにコマンドをコマンドラインで実行します。ドキュメントには、システムコールの概要、引数の説明、戻り値、エラーコードなどが記載されています。これらを参考に補足を加えた定義内容を以下に示します。

```c
{
    // - Parameters:
    //   - int: fd 入力は0、出力は1、エラーは2を指定
    //   - user_addr_t: cbuf
    //   - user_size_t: nbyte
    // - Returns:
    //   - user_ssize_t
    user_ssize_t write(int fd, user_addr_t cbuf, user_size_t nbyte);
}
```

シンタックスはC言語なので比較的読みやすいかと思います。これを見ると、 `write` 関数は3つの引数を受け取り、 `user_ssize_t` という型の戻り値を返すことがわかります。このように、日頃我々が使っている Swift のように、システムコールの呼び出し時には引数を渡すことができるのです。  
アセンブリでは、システムコールを実行する前にこれらの引数を明示的にレジスタに格納する必要があります。つまり、アセンブリコードを書く際には、これらの値を適切なレジスタにロードする必要があります。

```s
mov X0, #1
adr X1, greeting
mov X2, #16

greeting: .ascii "Hello, AkkeyLab\n"
```

上記のアセンブリコードは、 `write` システムコールの3つの引数をレジスタに格納するためのコードです。1行ずつ見ていきましょう。

```s
// オペコード
// ↓
  mov X0, #1
```

オペコードと呼ばれる命令には `mov` というローマ字が利用されています。これは、 `move` が語源で、値の移動を行う命令です。ここで、機械語に近い言語のわりには分かりやすい表現になっていることに気づくかと思います。  
これはニーモニック（Mnemonic）という表現で、CPU に対する命令を人間でも理解しやすい形にしたものです。機械語に変換されると、ここもビット表記になるのですが、これを人間が手書きしようとすると CPU の仕様書も読む必要が出てきます。

```s
  mov X0, #1
//    ~~~~~~
//       ↑
//    オペランド
```

オペコードの右側をオペランドと呼びます。このオペランドのシンタックスは CPU アーキテクチャ等によって異なります。今回ご紹介するのは `AArch64 アセンブリ記法` と呼ばれる記法（参考文献：*3）で `ARM アセンブリ記法` の 64bit 版です。詳細な用語説明は省略しますが、カンマを挟んで右から左にオペコードに準じた命令を実行してくれます。

```s
mov X0, #1    // レジスタ X0 に値 1 を設定する
```

これまでの解説を踏まえると、1行目は `1` を `X0` に `mov` すると読むことができます。 `mov` は値の移動を行う命令なので、レジスタの番地 `X0` に `1` を格納しているのだと分かります。これが `write` システムコールの第一引数に渡したい値で、今回は文字列の「出力」を行いたいので `1` を指定しています。  
なお、整数値の前にシャープを付けるのは、その数字がアドレス値ではないことを伝えるためのものです。

```s
adr X1, greeting // レジスタ X1 にラベル greeting のアドレスを設定する
```

2行目では `adr` という命令が利用されています。これは address の略で、シンボルのアドレスをレジスタへ格納するためのものです。ここでは `greeting` というラベルを指定しており、Swift における変数定義のように扱うことができます。つまり、 `greeting: .ascii "Hello, AkkeyLab\n"` で表示したい文字列をラベルを経由して指定していることが分かります。これが `write` システムコールの第二引数に渡したい値となります。

```s
mov X2, #16   // レジスタ X2 に値 16 を設定する
```

3行目は1行目と同じ `mov` 命令で、表示したい文字列の長さを指定しています。改行コードを1文字としてカウントしている点に注意しましょう。これが `write` システムコールの第三引数に渡したい値となります。

### 3. システムコールの実行
引数に渡す値をレジスタへ指定することができたので、実際にシステムコールを記述していきます。今まではレジスタの番地として `X0~2` を利用してきましたが、システムコールを行う時は特別な番地を使用する必要があります。

```s
mov X16, #4   // システムコールナンバーを 4 (write) に設定
svc #0x80     // Supervisor Call 命令でシステムコールを実行
```

システムコールを実行する際には、システムコールナンバーをレジスタ `X16` に設定します。ここでは、システムコールマスターファイルで確認した `4` を指定しているため、 `write` システムコールを実行するという意味になります。
`svc` 命令は「Supervisor Call」の略であり、システムコールを実行するための特別な割り込み命令です。この命令が実行されると、レジスタ `X16` に指定されたシステムコールナンバーに基づいて、カーネルが対応する処理を実行します。

### 4. プログラムの正常終了
プログラムの終了を、アセンブリでは明示的に記述する必要があります。終了処理にもシステムコールを使用するため、システムコールマスターファイルを確認します。該当箇所を以下に抜粋します。  

```
1	AUE_EXIT	ALL	{ void exit(int rval); }
```

この情報から、システムコールナンバーが `1` で、終了ステータスを引数として受け取ることが分かります。もちろん、詳細な仕様は `man exit` コマンドを利用して確認することができます。これを利用した終了処理を以下に示します。

```s
mov X0, #0   // 終了ステータスを 0 に設定
mov X16, #1  // システムコールナンバーを 1 (exit) に設定
svc #0x80    // システムコールを呼び出す
```

いかがでしょうか。最初は読めなかったアセンブリも読めるようになってきたように感じませんか？今回は正常終了させたいので終了ステータスは `0` です。  
以上で文字列の標準出力を行うプログラムが完成しました。プログラムのエントリーポイント指定などを追加して実際に動作するようにしたコードを以下に示します。

```s
.global _start
.align 2

_start:
    mov X0, #1 
    adr X1, greeting
    mov X2, #16
    mov X16, #4
    svc #0x80
    mov X0, #0
    mov X16, #1
    svc #0x80

greeting: .ascii "Hello, AkkeyLab\n"
```

### 5. `as` コマンドでアセンブルしてオブジェクトファイル（拡張子 `o`）を生成

```sh
vim greeting.s
as -arch arm64 \
    -o greeting.o greeting.s
```

ステップ4までで書いたアセンブリコードは拡張子が `s` のアセンブリファイルとして保存します。このファイルを `as` コマンドを利用してアセンブルすることでオブジェクトファイルを生成します。なお、今回 Apple Silicon で実行したいので、アーキテクチャ指定を arm64 としています。  
この段階では、まだリンクされていないので、生成されたオブジェクトファイルは実行不可能な中間ファイルとなります。

### 6. `ld` コマンドでリンクしてバイナリ（実行可能ファイル）を生成

```sh
# lSystem option: システムライブラリをリンク
# syslibroot option: SDK パスを指定
# e option: エントリーポイントを _start に設定
# arch option: アーキテクチャを arm64 に指定
ld -o greeting greeting.o \
    -lSystem \
    -syslibroot `xcrun -sdk macosx --show-sdk-path` \
    -e _start \
    -arch arm64
./greeting
```

最後に、 `ld` コマンドを使用してオブジェクトファイルをリンクし、実行可能なバイナリを生成します。そして、 `./greeting` コマンドを実行することで、コマンドライン上に文字列が出力され、正常に終了することを確認できます。

以上がアセンブリコードからバイナリファイルを生成し、実行するための手順です。必要に応じて `man` コマンドを使用してオプションの詳細を確認してください。

---

最後までお読みいただきありがとうございます。  
当初は計算処理も紹介予定でしたが、予想以上のボリュームになってしまったため、文字列の標準出力までを解説させていただきました。できる限り丁寧な説明を心がけましたが、非常に難易度の高い内容に感じた方もいたかもしれません。ですが、今年の iOSDC はオフライン開催が予定されておりますので、ぜひ私を探して質問攻めしちゃってください。  
また、本稿は GitHub で管理されておりますので、気軽に issue などの形で質問や修正依頼いただけますと幸いです。

## 参考文献
*1: [Furby Source Code](http://www.seanriddle.com/furbysource.pdf)   
*2: [システムコールマスターファイル](https://opensource.apple.com/source/xnu/xnu-1504.3.12/bsd/kern/syscalls.master)  
*3: [Apple Silicon から学ぶ CPU の歴史](https://github.com/AkkeyLab/technical-article/blob/main/cpu-history.md)

## 著者
Akio Itaya (akkey)

- AkkeyLab株式会社 代表取締役
- 株式会社AppBrew エンジニア
- 合同会社アイネット エンジニア

本稿執筆時点で3社に所属するプログラマー兼経営者。  
緑の毛色が特徴の初代ファービーを両親に買ってもらって遊んだのが良い思い出。遊んでいる間に何をしてもすぐに寝てしまうようになってしまい、乾電池を抜いてしまった。アセンブリを読めば、なぜすぐ寝てしまうのか解読できるのかもしれませんね。

AkkeyLab は 板谷 晃良 の商標又は登録商標です
