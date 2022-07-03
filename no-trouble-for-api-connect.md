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

引用：https://swagger.io/specification/

OpenAPI Specification には上記のように説明が書かれており、スマートフォンアプリエンジニア目線で見ると、API 仕様書フォーマットの1つであるということがわかります。つまり、この仕様書がバックエンドとフロントエンドで共通のインターフェースとなるのです。

累計数百万枚ものレシート買取実績を持つ ONE の開発チームには、仕様を確定させた後に以下のような手順で開発を進める文化があります。なお、説明の都合上、以後スマートフォンアプリエンジニアをフロントエンドエンジニアと表現することとします。

1. バックエンドもしくはフロントエンドエンジニアが OpenAPI 定義を更新
2. 施策に関わるバックエンドとフロントエンド両者の承認を得たプルリクがマージされる
3. バックエンド、フロントエンド エンジニアでそれぞれ開発を開始

ポイントは、バックエンドとフロントエンドエンジニアの両者が OpenAPI 管理に関わるという点です。このように OpenAPI が共通言語となっているため、バックエンド・フロントエンドがどんな言語・設計で構築されているかに関係なく両者が API 仕様の議論に集中できるのです。  
このように API 仕様を管理することで、冒頭で述べた失敗事例の殆どを克服することができると言っても良いかもしれません。しかし、仕様が決まったとしても、実際に API をコールすることができるのはバックエンドの開発がある程度進んでからになってしまいます。このようなタイミングでも快適に開発を進めるテクニックを次に紹介していくことにしましょう。

## XcodePreviews 活用術
SwiftUI とセットでリリースされた機能である XcodePreviews ですが、副業では特に多くの恩恵を得ることができる神機能だと思っています。


## Moya 利用時の Stub 活用術