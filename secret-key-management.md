# シークレットキー管理と安全な開発手法

## Introduction
皆さんはシークレットキーをどのように管理していますか？
GitHub Sponsors を利用することで、一定の第三者にコードを公開する手法が登場するなど、ソースコードの公開スタイルは様々です。ただし、機密情報の管理方法には注意が必要です。契約を結んでいない技術者に機密情報が見える状態は、漏洩や不正アクセスなどのトラブルを引き起こす可能性を高めます。
本稿では、既存のチーム開発スタイルを維持しつつ、機密情報を Git のコミットに含めない開発手法を紹介します。Xcode Cloud への対応はもちろん、KMM（Kotlin Multiplatform Mobile）などの異なる開発スタイルでも有効な方法です。これは特に OSS活動 で力を発揮しますが、業務委託契約など、多様な働き方が進む中でトラブルを未然に防ぐためのノウハウとして活用できます。

## 前提
本稿では事例の一つとしてとある API Key を取り扱います。しかし、場合によっては本稿で紹介する方法が最善策ではない可能性がありますのでご注意ください。  
例えば、API Key がコード上になかったとしても配布したアプリの通信を解析したり、アプリを逆コンパイルしたりすることで API Key を取得することが可能な場合があるからです。この場合、バックエンド側の実装に隠蔽することが最善策と言えるでしょう。

本稿は GitHub で管理されており、最新の状態が閲覧できるだけでなく、皆様からのご指摘にも対応可能となっております。ぜひ、ご活用ください。  
GitHub: https://github.com/AkkeyLab/technical-article

## 方針
1. info.plist に定義のみ追加
2. Keychain Access にシークレットキーを追加
3. ビルドのタイミングで info.plist に書き込み

まずは、KMMなどを活用しない通常の環境下でのソリューションをご紹介します。手順は上記の3ステップになります。

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

上記のコードは、 `info.plist` ファイルにシークレットキーの定義を環境変数として追加するものです。ここでは、国税庁の公式 API を使用するシステムを例に説明しますが、実際に適用する際には適切な値に置き換えてください。  
このコードでは、 `LSEnvironment` キーの下に `NATIONAL_TAX_AGENCY_API_KEY` キーを追加し、その値を `$(NTA_API_KEY)` に設定しています。 `$(NTA_API_KEY)` は後で正しい値に置き換えられるため、適当な値で構いません。

### 2. Keychain Access にシークレットキーを追加
![image](images/how-to-hide-the-key-029.png)

シークレットキーをローカルマシンの Keychain Access アプリに保存するためには、GUI または `security` コマンドを使用します。チーム開発環境では、1password などのパスワード管理ソフトウェアにシークレットキーと `security` コマンドをメモしておくと便利です。また、 `security` コマンドの詳細は、`man security` コマンドを使用することで参照できます。  
ただし、CI（継続的インテグレーション）ではこのコマンドを実行しないでください。ちなみに、Xcode Cloud では `security` コマンドの使用が禁止されています。

### 3. ビルドのタイミングで info.plist に書き込み
まずはじめに、仮の定義を行った `info.plist` ファイルに正しいシークレットキーを書き込みましょう。ただし、直接 `info.plist` ファイルを書き換えてしまうと、ファイルに差分が生じてしまいます。そのため、ビルド時に一時ファイルとして生成される `Preprocessed-Info.plist` ファイルを書き換える手法を採用します。  
一時ファイルの生成はデフォルトで無効化されているため、「Build Settings」から「Preprocess Info.plist File」という項目の値を YES に変更することで有効化します。

```xml
F4C37D3F296AEE2200D0084B /* Debug */ = {
	isa = XCBuildConfiguration;
	buildSettings = {]
		INFOPLIST_PREPROCESS = YES;
	};
	name = Debug;
};
```

上記は `project.pbxproj` ファイル内で定義されている該当箇所の一部です。ただし、「Build Configuration」の内容や個数はプロジェクトによって異なる可能性があるため、シークレットキーを使用する全ての部分に適用してください。

次に重要なのは、 `Preprocessed-Info.plist` ファイルを書き換えるタイミングです。このファイルは一時ファイルであるため、利用される直前に書き換えるのが最適です。したがって、plist ファイルを書き換えるスクリプトを「Copy Bundle Resources」の直前に配置するのが良いと言えます。  
実際にスクリプトを設定したときの `project.pbxproj` ファイルの一部を以下に示します。

```xml
/* Begin PBXNativeTarget section */
		F4C37D2F296AEE2100D0084B /* c-search */ = {
			...
			buildPhases = (
				F4C37D2C296AEE2100D0084B /* Sources */,
				F4C37D2D296AEE2100D0084B /* Frameworks */,
				F4FDC7E729979DDB0057D80A /* Setting Environment Variables */,
				F4C37D2E296AEE2100D0084B /* Copy Bundle Resources */,
				F4FDC7FB2998D9A10057D80A /* Embed App Clips */,
				F46CEF4E29BD9D610096B6E1 /* Embed Foundation Extensions */,
			);
			...
		};
/* End PBXNativeTarget section */
```

上記のコードは、「Build Phases」の設定が記述されている箇所で、 `Copy Bundle Resources` の直前に `Setting Environment Variables` という名前のスクリプトが配置されていることが分かります。これにより、ビルド時にスクリプトが実行され、 `Preprocessed-Info.plist` ファイルが書き換えられます。その後、変更内容が正しく反映された plist ファイルがバンドルリソースにコピーされます。  
ただし、プロジェクトによって `project.pbxproj` ファイルの構造や設定の名称が異なる場合がありますので、適切な箇所にスクリプトを配置するように調整してください。

```xml
/* Begin PBXShellScriptBuildPhase section */
		F4FDC7E729979DDB0057D80A /* Setting Environment Variables */ = {
			...
			inputPaths = (
				"$(TEMP_DIR)/Preprocessed-Info.plist",
			);
			...
			shellPath = /bin/sh;
			shellScript = "...";
		};
/* End PBXShellScriptBuildPhase section */
```

上記のコードは、先程設定した `Setting Environment Variables` という名前のスクリプトの詳細設定を示しています。特に、 `Preprocessed-Info.plist` ファイルのパスを「Input Paths」に追加する必要がある点に注意してください。  
ここまでで、 `info.plist`を書き換えるための準備が整いました。次に、シークレットキーを書き込むスクリプトを書くことにしましょう。

```sh
# a option: キーチェーン内のアカウント名を指定
# s option: キーチェーン内のサービス名（またはシークレットキーの識別子）を指定
# w option: 検索されたシークレットキーの値を出力
security find-generic-password \
    -a NATIONAL-TAX-AGENCY \
    -s NTA-API-KEY \
    -w
```

まず、Keychain Access に保存されたシークレットキーを取得する方法を確認しましょう。実は非常にシンプルで、保存時と同じ `security` コマンドを使用して取得できます。上記のスクリプトは、 `security` コマンドを使用してシークレットキーを取得する一例です。各オプションに指定する値は適宜実際の値に置き換えてください。  
なお、シークレットキーの取得処理は実行環境に依存するため、適切な権限や認証情報が必要な場合があります。また、必要に応じて `security` コマンドの他のオプションを使用して設定を調整することもできます。

```sh
/usr/libexec/PlistBuddy -c \
    "Set:LSEnvironment:NATIONAL_TAX_AGENCY_API_KEY ${NTA_API_KEY}" \
    "${TEMP_DIR}/Preprocessed-Info.plist"
```

次に、plist ファイルの編集には `PlistBuddy` コマンドを使用します。このコマンドは Xcode Cloud でも利用できます。  
上記のスクリプトでは、 `-c` オプションの後に続く1つ目の文字列で編集内容を指定します。この場合、 `Set:` を使って値の書き込みを指定し、編集対象のキーを `LSEnvironment:NATIONAL_TAX_AGENCY_API_KEY` と指定しています。 `${NTA_API_KEY}` はシェル変数であり、後ほど詳しく解説します。  
2つ目の文字列では、編集対象の plist ファイルのパスを指定しています。`${TEMP_DIR}` はシェル変数であり、ビルド時に準備される一時ファイルのディレクトリパスが指定されます。

このスクリプトを実行することで、指定したキーに対してシークレットキーが `Preprocessed-Info.plist` ファイルに書き込まれます。なお、編集するキーのパスはプロジェクトの要件に合わせて適切に設定してください。

```swift
guard let environment = Bundle.main.object(forInfoDictionaryKey: "LSEnvironment") as? [String: String],
      let apiKey = environment["NATIONAL_TAX_AGENCY_API_KEY"],
      !apiKey.isEmpty
else {
    throw  SearchCompanyError.apiKeyNotFound
}
```

最後に、plist からシークレットキーを取得する処理を iOS アプリに実装し、動作確認を行ってみましょう。上記のコードでは、 `Bundle.main.object(forInfoDictionaryKey: "LSEnvironment")` を使用して、 `info.plist` ファイルから LSEnvironment の値を取得しています。その後、取得した値を辞書型としてキャストし、シークレットキーの値を取り出しています。  
完全なスクリプトは「Xcode Cloud 対応」セクションに載せていますので、実践される際はそちらもご確認ください。

## Xcode Cloud 対応
Xcode Cloud では `security` コマンドを使用することができないため、別の方法でシークレットキーを取得する必要があります。セキュアな管理を実現するため、環境変数を活用してみましょう。

```sh
if [ ! "${NTA_API_KEY}" ];then
    export NTA_API_KEY="$(security find-generic-password -a NATIONAL-TAX-AGENCY -s NTA-API-KEY -w)"
fi
```

上記のようにスクリプトを改良することで、ローカルマシンでは Keychain Access を使用し、Xcode Cloud では環境変数を使用してシークレットキーを取得できるようになります。ここでは Xcode Cloud を例に説明していますが、他の CI サービスでも同様の環境変数を使用してシークレットキーを管理することができます。  
このスクリプトでは、まず環境変数 `${NTA_API_KEY}` が設定されているかを確認しています。もし設定されていない場合は、 `security` コマンドを使用して Keychain Access からシークレットキーを取得し、環境変数に設定しています。  
最終的なスクリプトは以下のようになります。

```sh
if [ ! "${NTA_API_KEY}" ];then
    export NTA_API_KEY="$(security find-generic-password -a NATIONAL-TAX-AGENCY -s NTA-API-KEY -w)"
fi
if [ ! "${NTA_API_KEY}" ];then
    echo "error: NTA_API_KEY does not exist in Keychain Access"
fi
/usr/libexec/PlistBuddy -c \
    "Set:LSEnvironment:NATIONAL_TAX_AGENCY_API_KEY ${NTA_API_KEY}" \
    "${TEMP_DIR}/Preprocessed-Info.plist"
```

このように改良を行うことで、Xcode Cloud などの環境でもセキュアにシークレットキーを管理することができます。

## KMM 対応
KMM（Kotlin Multiplatform Mobile）は、ビジネスロジックを Kotlin で記述し、iOS と Android の両方で使用できる仕組みです。KMM では UI 設計が共通化されていない点で他のマルチプラットフォームと異なった特徴を持っています。  
シークレットキーを利用する処理は、通常ビジネスロジックに含まれることが多いため、これまでに紹介した方法を KMM にそのまま適用するのは難しいかもしれません。そのため、KMMでは以下の2つのステップでシークレットキーの管理環境を整えることにします。

1. サードパーティライブラリの導入
2. シークレットキーを環境変数に追加

### 1. サードパーティライブラリの導入
Android アプリ開発では、Gradle を使用してビルドする際に `BuildConfig` というクラスが生成されます。このクラスは、デバッグとプロダクションなどの環境ごとの変数を定義するために使用され、Xcode の「Build Configuration」に類似した役割を果たします。  
もし `BuildConfig` クラスでシークレットキーを取得・定義できれば、アプリのコードから直接シークレットキーを参照せずに済むため、安全性を向上させることができるでしょう。しかし、本稿執筆時点で KMM では `BuildConfig` のような機能が実装されていません。  
そこで、KMM でも同様の機能を実現できるサードパーティライブラリの導入を検討してみましょう。

```kotlin
plugins {
    id("com.codingfeline.buildkonfig").version("+")
}
```

上記のコードは、yshrsmzさんが提供する `BuildKonfig` （参考文献：*1）を導入するための設定です。KMM プロジェクトの `build.gradle.kts` ファイルにこのコードを追加することで、 `BuildKonfig` を利用できるようになります。

```kotlin
buildkonfig {
    packageName = "..."

    defaultConfigs {
        buildConfigField(STRING, "NTA_API_KEY", "key")
    }

    targetConfigs {
        create("android") {
            buildConfigField(STRING, "NTA_API_KEY", "key")
        }
        create("ios") {
            buildConfigField(STRING, "NTA_API_KEY", "key")
        }
    }
}
```

上記のコードは、 `BuildKonfig` を活用した設定の一例です。 `buildkonfig` ブロックを使用して、 `defaultConfigs` と `targetConfigs` のブロックがネストされて定義されています。これにより、iOS と Android で異なる値を定義することができます。  
最も重要な部分は、 `buildConfigField` 関数です。第一引数に変数の型を指定し、第二引数に変数名を文字列で指定します。そして、第三引数には実際のシークレットキーを文字列で指定します。ただし、直接シークレットキーを記述すると、コミットにシークレットキーが含まれてしまう可能性があるため、適切なセキュリティ対策が必要です。

### 2. シークレットキーを環境変数に追加

```kotlin
System.getenv("NTA_API_KEY")
```

ローカルの環境変数を取得する処理を Kotlin で実装すると上記のようになります。この方法を使うと、特に追加の対応を必要とせずに、ローカル環境や CI 環境での動作が可能となります。

```kotlin
buildConfigField(STRING, "NTA_API_KEY", "${System.getenv("NTA_API_KEY")}")
```

環境変数を取得する処理を反映させた `buildConfigField` 関数を上記に示します。

```sh
echo -e "export NTA_API_KEY=8440F6C4-EB...B1" >> ~/.zshrc
```

次に、ローカル環境で環境変数を管理する方法について説明します。もっとも簡単な方法は、上記のように `.zshrc` など、お使いのシェルの RC ファイルに直接記述する方法です。ただし、この方法ではローカル環境内でどこからでも簡単にアクセスできてしまうため、セキュリティ上の懸念があるかもしれません。その場合、 `direnv` というツール（参考文献：*2）を使用することをおすすめします。このツールを使うと、ディレクトリごとに異なる環境変数を設定できるようになります。

```kotlin
val apiKey = BuildKonfig.NTA_API_KEY
val response: HttpResponse = client.get("https://api....nta.go.jp/../") {
    parameter("id", apiKey)
    ...
}
```

最後に、プログラム内で環境変数にアクセスするプログラムは上記のようになります。

---

最後までお読みいただきありがとうございます。  
SDK によってはシークレットキーだけでなく、アプリの「Bundle Identifier」などを使って認証しているため、そこまで神経質になる必要はないのかもしれません。しかし、セキュリティに関して考えることには大きな意味があると私は考えています。  
どこか一部でも皆さんの環境に活かせるものがあると嬉しいです。  
本稿は GitHub で管理されておりますので、気軽に issue などの形で質問や修正依頼いただけますと幸いです。

## 参考文献
*1: https://github.com/yshrsmz/BuildKonfig  
*2: https://github.com/direnv/direnv

## 著者
Akio Itaya (akkey)

- AkkeyLab株式会社 代表取締役
- 株式会社AppBrew エンジニア
- 合同会社アイネット エンジニア

本稿執筆時点で3社に所属するプログラマー兼経営者。  
Swift との出会いは大学。まだ公式ドキュメント以外の情報が少なかったこともあり、Objective-C で書かれたコードを頑張って読んでいたのが良い思い出。今でも Swift を書いてます。

AkkeyLab は 板谷 晃良 の商標又は登録商標です
