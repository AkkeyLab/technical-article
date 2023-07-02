# シークレットキー管理と安全な開発手法

## Introduction
皆さんはシークレットキーをどのように管理していますか？
GitHub Sponsors を利用することで、一定の第三者にコードを公開する手法が登場するなど、ソースコードの公開スタイルは様々です。ただし、機密情報の管理方法には注意が必要です。契約を結んでいない技術者に機密情報が見える状態は、漏洩や不正アクセスなどのトラブルを引き起こす可能性を高めます。
本稿では、既存のチーム開発スタイルを維持しつつ、機密情報を Git のコミットに含めない開発手法を紹介します。Xcode Cloud への対応はもちろん、KMM（Kotlin Multiplatform Mobile）などの異なる開発スタイルでも有効な方法です。これは特に OSS活動 で力を発揮しますが、業務委託契約など、多様な働き方が進む中でトラブルを未然に防ぐためのノウハウとして活用できます。

本稿は GitHub で管理されており、最新の状態が閲覧できるだけでなく、皆様からのご指摘にも対応可能となっております。ぜひ、ご活用ください。  
GitHub: https://github.com/AkkeyLab/technical-article

## 前提
本稿では事例の一つとしてとある API Key を取り扱います。しかし、場合によっては本稿で紹介する方法が最善策ではない可能性がありますのでご注意ください。  
例えば、API Key がコード上になかったとしても配布したアプリの通信を解析したり、アプリを逆コンパイルしたりすることで API Key を取得することが可能な場合があるからです。この場合、バックエンド側の実装に隠蔽することが最善策と言えるでしょう。

## 方針
1. info.plist に定義のみ追加
2. Keychain Access にシークレットキーを追加
3. ビルドのタイミングで info.plist に書き込み

まずは、KMM などを活用しないバニラ環境下におけるソリューションをご紹介します。手順は上記3ステップとなります。

### 1. info.plist に定義のみ追加

```xml
<plist version="1.0">
<dict>
	<key>LSEnvironment</key>
	<dict>
		<key>NATIONAL_TAX_AGENCY_API_KEY</key>
		<string>$(NTA_API_KEY)</string>
	</dict>
</dict>
</plist>
```

info.plist ファイルにシークレットキーの定義を環境変数として追加します。この時、value は後ほど正しい値に置換されるので、適当な値もしくは空にしておきます。  
なお、今回は国税庁公式 API を使ったシステムを例にご紹介しますので、実践される場合は key や value の値を適宜書き換えてください。

### 2. Keychain Access にシークレットキーを追加
![image](images/how-to-hide-the-key-029.png)

GUI もしくは `security` コマンド経由で、ローカルマシンの Keychain Access App にシークレットキーを保存します。チーム開発環境下では、1password などのパスワード管理ソフトにシークレットキーと共に登録コマンドをメモしておくと便利かもしれません。  
なお、 `security` コマンドについては `man security` で調べることができます。

ただし、この操作を CI などで実施しないでください（Xcode Cloud では `security` コマンドの使用が禁じられています）。なぜなら、実行コマンドをセキュアに管理することができないからです。

### 3. ビルドのタイミングで info.plist に書き込み

```sh
security find-generic-password -a NATIONAL-TAX-AGENCY -s NTA-API-KEY -w
```

まず初めに、Keychain Access に保存されたシークレットキーを取得する方法を確認しておきましょう。実は非常にシンプルで、保存時と同じ `security` コマンドで取得が可能です。

```xml
F4C37D3F296AEE2200D0084B /* Debug */ = {
	isa = XCBuildConfiguration;
	buildSettings = {]
		INFOPLIST_PREPROCESS = YES;
	};
	name = Debug;
};
F4C37D40296AEE2200D0084B /* Release */ = {
	isa = XCBuildConfiguration;
	buildSettings = {
		INFOPLIST_PREPROCESS = YES;
	};
	name = Release;
};
```

次に、plist ファイルの編集には `PlistBuddy` コマンドを利用できます。  


```sh
/usr/libexec/PlistBuddy -c "Set:LSEnvironment:NATIONAL_TAX_AGENCY_API_KEY ${NTA_API_KEY}" "${TEMP_DIR}/Preprocessed-Info.plist"
```

```sh
if [ ! "${NTA_API_KEY}" ];then
    export NTA_API_KEY="$(security find-generic-password -a NATIONAL-TAX-AGENCY -s NTA-API-KEY -w)"
fi
if [ ! "${NTA_API_KEY}" ];then
    echo "error: NTA_API_KEY does not exist in Keychain Access"
fi
/usr/libexec/PlistBuddy -c "Set:LSEnvironment:NATIONAL_TAX_AGENCY_API_KEY ${NTA_API_KEY}" "${TEMP_DIR}/Preprocessed-Info.plist"
```

---

最後までお読みいただきありがとうございます。

どこか一部でも皆さんの環境に活かせるものがあると嬉しいです。  
本稿は GitHub で管理されておりますので、気軽に issue などの形で質問や修正依頼いただけますと幸いです。
