# API 繋ぎこみトラブル撲滅委員会

## Introduction
API 繋ぎこみを後回しにした経験、皆さんありますでしょうか。  
もしあるのであれば、開発終盤で API の仕様変更が必要なことに気づいたり、API 仕様とシンクしない UI を構築してしまったり、バックエンド側の開発速度に強く依存してしまうという経験をお持ちかもしれません。特に最後のものに関しては「仕方ないよね」と根本的解決を置き去りにされてしまう可能性が高いように思います。

本稿では、私が本業と複数の副業で実践済みのテクニックをご紹介します！  
実例と共に初心者〜中級者向けに執筆いたしました。大規模なリファクタリング不要で、読んでいただいた直後から実践・成果を実感していただけるはずです！

本稿は GitHub で管理されており、最新の状態が閲覧できるだけでなく、皆様からのご指摘にも対応可能となっております。ぜひ、ご活用ください。  
GitHub: https://github.com/AkkeyLab/technical-article

## アプリ開発者目線で見る OpenAPI
まず、OpenAPI が一体何者なのかから見ていくことにしましょう。もしかしたら、Swagger と言うと聞いたことがあるという方も多いかもしれません。

> The OpenAPI Specification (OAS) defines a standard, language-agnostic interface to RESTful APIs which allows both humans and computers to discover and understand the capabilities of the service without access to source code, documentation, or through network traffic inspection. When properly defined, a consumer can understand and interact with the remote service with a minimal amount of implementation logic.

引用： https://swagger.io/specification/

OpenAPI Specification には上記のように説明が書かれており、スマートフォンアプリエンジニア目線で見ると、API 仕様書フォーマットの1つであるということがわかります。つまり、この仕様書がバックエンドとフロントエンドで共通のインターフェースとなるのです。

累計数百万枚ものレシート買取実績を持つ ONE の開発チームには、仕様を確定させた後に以下のような手順で開発を進める文化があります。なお、説明の都合上、以後スマートフォンアプリエンジニアをフロントエンドエンジニアと表現することとします。

1. バックエンドもしくはフロントエンドエンジニアが OpenAPI 定義を更新
2. 施策に関わるバックエンドとフロントエンド両者の承認を得たプルリクがマージされる
3. バックエンド、フロントエンド エンジニアでそれぞれ開発を開始

ポイントは、バックエンドとフロントエンドエンジニアの両者が OpenAPI 管理に関わるという点です。このように OpenAPI が共通言語となっているため、バックエンド・フロントエンドがどんな言語・設計で構築されているかに関係なく両者が API 仕様の議論に集中できるのです。  
このように API 仕様を管理することで、冒頭で述べた失敗事例の殆どを克服することができると言っても良いかもしれません。しかし、仕様が決まったとしても、実際に API をコールすることができるのはバックエンドの開発がある程度進んでからになってしまいます。このようなタイミングでも快適に開発を進めるテクニックを次に紹介していくことにしましょう。

## Moya 利用時の Stub 活用術
ネットワーク抽象化レイヤ Moya を皆さんはご存知でしょうか。  
Alamofire に依存する OSS で、RxSwift などとも依存関係を持つため賛否はあるものの、個人的にも愛用しているサードパーティ製 framework です。

### 前提
次に、Stub（スタブ）をご存知でしょうか。  
プログラミングの分野では、実際の動作結果の代わりに準備された代用品という意味で使われる用語になります。例えば、iOS アプリの単体テストで実際にサーバと通信を行った場合、オフライン環境やサーバメンテ中の環境ではテスト不合格という結果になってしまいます。そこで、通信結果を Stub という形で準備することで、通信環境やサーバの状態に依存することなく iOS アプリテストに集中できるのです。  
Stub は知ってた！という皆さんは Stub と Mock の違いについて考えてみると面白いかもしれません。ぜひ iOSDC 会場での会話ネタにしてみてください！  

### Stub の利用
では、Moya でその Stub を利用する手順を見ていきましょう。  
API コールに必要な情報を Moya では TargetType というプロトコルに準拠させる形で定義して利用します。以下に実際の Moya 内部のコードを示します（本稿執筆時点のコード）。

```swift
import Foundation

public protocol TargetType {
    var baseURL: URL { get }
    var path: String { get }
    var method: Moya.Method { get }
    var sampleData: Data { get }
    var task: Task { get }
    var validationType: ValidationType { get }
    var headers: [String: String]? { get }
}
```
引用： https://github.com/Moya/Moya/blob/master/Sources/Moya/TargetType.swift

上記の、Data 型を返す sampleData に、通信に成功したと場合のレスポンスを指定することで、通信処理で Stub を利用することができるようになります。具体的な実装方法を以下に示します。

```swift
var sampleData: Data {
    """
    [
        {
            "id": "fcf81b2a-36d2-d695-d4b2-akkey2022lab",
            "name": "iOSDC Japan 2022"
        }
    ]
    """.data(using: .utf8)!
}
```

Moya では以下のように Stub を利用するように指定することで、実際に通信を行わず、先程定義した sampleData を結果として返してくれるようになります。汎用的な実装方法に関しては、Moya プロジェクトの単体テストを参考にしてみると良いでしょう。

```swift
struct SampleTarget: TargetType { ... }

MoyaProvider<SampleTarget>(stubClosure: MoyaProvider.immediatelyStub)
    .request(
        SampleTarget(),
        completion: { result in
            // do something
        }
    )
```

### 応用
ここまで Stub とそれを Moya で利用するための方法を紹介してきました。これらは単体テストで利用されることが多いですが、バックエンド開発が終わっていないタイミングの開発でも力を発揮してくれるのです。  

```swift
#if DEBUG
    MoyaProvider<SampleTarget>(stubClosure: MoyaProvider.immediatelyStub)
#else
    MoyaProvider<SampleTarget>()
#endif
    .request(
        SampleTarget(),
        completion: { result in
            // do something
        }
    )
```

上記のように分岐処理を施すことで我々が開発しやすくなるだけでなく、チームでのデザインレビューをバックエンド開発を待たずして実施することが可能になります。

このように Stub を利用することでバックエンドの開発に依存することなく API 通信に関連する処理が実装可能なことを紹介しました。しかし、通信結果を表示する UI の実装に時間がかかってしまえば、動作確認する頃には優秀なバックエンドエンジニアが実装を終えていることでしょう。無駄ではないですが、利点を最大限活かしているとは言い難いのが現状です。  
そこで、UI の実装時間を短縮する方法を考えてみましょう。

## XcodePreviews 活用術
SwiftUI とセットでリリースされた機能である XcodePreviews ですが、副業では特に多くの恩恵を得ており、個人的に神機能として愛用しているものの一つです。

### 前提
エックスコードプレビュー？ナニソレオイシイノって方向けに要点をまとめると以下のようになります。

- コードで記述したレイアウトをリアルタイムに表示してくれる
- Storyboard や xib で有名な Interface Builder（IB）のように視覚的にアプリ画面を設計するための補助ツール

IB は NEXTSTEP 時代から存在するソフトウェアが元になっており、当初は独立したソフトウェアとして開発されていました。そして、Xcode 4.0 に統合されて現代まで利用されてきたという過去を持っています。  
そんな IB に変わる存在として XcodePreviews が登場したのです。こんな歴史的瞬間だからこそ、先駆者として XcodePreviews を使い倒してみませんか？

きっと、XcodePreviews が気になりすぎて夜も眠れない状態になってしまった読者の方もいることでしょう。安心してください、どのような原理で動作しているのか GitHub Gist 上で執筆しておりますので、是非ご覧ください。  
GitHub Gist: https://gist.github.com/AkkeyLab/67af9a91498c6c5ad138123cb8ae5c28

### 課題
iOS アプリ開発では、複数の要素で成り立つ画面において「途中経過を視覚的に共有しにくい」という状況に遭遇しやすいと感じています。これによって、進捗共有で「まだ見せられるものはありませんが◯◯◯の実装を行いました」といったコミュニケーションが生まれてしまいます。  
副業は特に働く場所・時間など自由度が高いことが多いです。ですから、先程のようなコミュニケーションを避けることは 開発メンバー・委託元との信頼構築 を行う中で最も重要なことであると私は考えています。

### 実例
![xcode-previews-sample](images/xcode-previews-sample.png)

XcodePreviews の利用イメージを上記に示します。このようにコンポーネント単位で実行可能で、各デバイス毎のシミュレート結果が開発画面右側に表示されます。これを進捗共有ミーティングで見せています。また、画面左側のプログラムを書き換えると瞬時にシミュレート結果に反映されるため、ステータスの変更もスムーズに共有可能です。  
ということで、実際に業務で利用しているものをお見せしたかったのですが、残念ながら叶いませんでした。なので今回は、私が OSS として公開しているプロジェクトでの利用イメージを掲載させていただきました。  
GitHub: https://github.com/AkkeyLab/LazyGrid

---

最後までお読みいただきありがとうございます。  
本稿は GitHub で管理されておりますので、気軽に issue などの形で質問や修正依頼いただけますと幸いです。
