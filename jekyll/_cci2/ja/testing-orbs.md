---
layout: classic-docs
title: "Orb のテスト手法"
short-title: "テスト手法"
description: "CircleCI Orbs のテストに関する入門ガイド"
categories:
  - getting-started
order: 1
version:
  - Cloud
---

Orbs のテストに利用できるさまざまな手法について説明します。

- 目次
{:toc}

## はじめに

CircleCI Orbs とは構成のエレメントをまとめたパッケージです。構成時に Orbs を使用することで、ワークフローを簡素化し、ワークフロー内でアプリケーションや Orbs をすばやく簡単にデプロイできます。 ワークフローの Orb を作成したら、Orb をデプロイしてパブリッシュする前に、Orb をテストして、一定のニーズを満たしているかどうかを確認できます。

Orb のテストには 4 つのレベルがあります。レベルが上がるごとに内容が複雑になり、その範囲も広がります。

- スキーマ バリデーション - このテストは、1 つの CLI コマンドで実行可能です。Orb の YAML の形式が正しいかどうか、Orb スキーマに準拠しているかどうかを確認します。
- 展開テスト - CircleCI CLI のスクリプトを記述して実行します。このテストでは、Orb のエレメントを含む設定が処理されるときに、意図したとおりの設定が生成されるかどうかを確認します。
- 実行時テスト - このテストを実行するには、別のテストを準備し、それらを CircleCI ビルド内で実行する必要があります。
- インテグレーション テスト - 通常このテストは、サードパーティ サービスとの安定したパブリック インターフェースとして特別に設計された Orbs など、きわめて高度な Orbs でのみ必要になります。 Orb のインテグレーション テストを実行するには、インテグレーション先のシステムに合わせて、カスタム ビルドと独自の外部テスト環境を準備する必要があります。

ここからは、Orb テストの方法についてレベル別に詳しく説明します。

### スキーマ バリデーション

Orb が有効な YAML であり、スキーマに従った正しい形式になっているかどうかをテストするには、CircleCI CLI で `circleci orb validate` を使用します。

#### 例

たとえば Orb のソースが `./src/orb.yml` にある場合、`circleci orb validate ./src/orb.yml` を実行することで、Orb が有効かどうか、設定ファイルの処理が正しく進むかどうかについて、フィードバックを受け取ることができます。 エラーが発生した場合は、最初のスキーマ バリデーション エラーが返されます。 また、ファイル パスではなく STDIN を渡しても実行できます。

たとえば、上記の例なら、`cat ./src/orb.yml | circleci orb validate -` として実行します。

**メモ**: スキーマ エラーはよく「裏こそ表」と表現されます。コーディング エラーの内容を把握するには、エラー スタックの最も内側のエラーを見るのが一番です。

バリデーション テストには、CircleCI CLI を使用してビルド内で実行する方法もあります。 `circleci/circleci-cli` Orb を直接使用して、CLI をビルドに挿入できます。また、ジョブ内でスキーマ バリデーションをデフォルトで実行するなどの便利なコマンド ジョブをいくつか含む `circleci/orb-tools` Orb を使用することも可能です。 `orb-tools` を使用して基本的なパブリッシュを実行する Orb には "hello-build" Orb などがあります。

### 展開テスト

次のレベルの Orb テストでは、Orb が展開されて、CircleCI システムで使用される最終的な `config.yml` が意図したとおりに生成されるかどうかをバリデーションします。

このテストを実行するときには、Orb を dev バージョンとしてパブリッシュし、それを設定ファイルで使用して処理するか、設定ファイルをパッケージ化してインライン Orb にしたうえで処理することをお勧めします。 次に、`circleci config` プロセスを使用し、意図していた展開状態と実際の結果を比較します。

Orb が別の Orbs を参照している場合は、別の形式の展開テストを対象の Orb に実行できます。 `circleci orb` プロセスを使用すると、Orbs に依存している Orbs が解決でき、レジストリにパブリッシュしたときにどのように展開されるかを確認できます。

なお、展開をテストするときは、展開される文字列リテラルではなく、データの基底構造をテストすることが肝心です。 YAML 内の記述についてアサーションする場合は、`yq` を使用すると便利です。 このツールを使用すると、特定の構造エレメントをチェックできます。文字列の比較や展開後のジョブの各部に依存しないため、テストが不安定になることもありません。

以下に、CLI から基本的な展開テストを実行する手順を示します。

1) `src/orb.yml` にあるシンプルな Orb を想定します。

{% raw %}
```yaml
version: 2.1

executors:
  default:
    parameters:
      tag:
        type: string
        default: "curl-browsers"
      docker:

        - image:  circleci/buildpack-deps:parameters.tag

jobs:
  hello-build:
    executor: default
    steps:

      - run: echo "Hello, build!"
```
{% endraw %}

2) `circleci orb validate src/orb.yml` を使用して Orb をバリデーションします。

3) `circleci orb publish src/orb.yml namespace/orb@dev:0.0.1` を使用して dev バージョンをパブリッシュします。

4) その Orb を `.circleci/config.yml` に入れます。

{% raw %}
```yaml
version: 2.1

orbs:
  hello: namespace/orb@dev:0.0.1

workflows:
  hello-workflow:
    jobs:

      - hello/hello-build
```
{% endraw %}

`circleci config process .circleci/config.yml` を実行すると、以下のような結果が表示されます。

{% raw %}
```yaml
version: 2.1

jobs:
  hello/hello-build:
    docker:

      - image: circleci/buildpack-deps:curl-browsers
    steps:
      - run:
          command: echo "Hello, build!"
workflows:
  hello-workflow:
    jobs:
      - hello/hello-build
  version: 2
```
{% endraw %}

`config.yml` ファイルは以下のようになります。

{% raw %}
```yaml
version: 2.1

orbs:
  hello: namespace/orb@dev:0.0.1

  workflows:
    hello-workflow:
    jobs:

        - hello/hello-build
```
{% endraw %}

これで、上記の結果をカスタム スクリプトで使用して、その構造を意図する構造と比較するテストを実行できます。 This form of testing is useful for ensuring you do not break any contracts your orb interface has established with users of the orb, while also enabling you to test different parameter inputs to determine how these parameter inputs impact what is generated during config processing.

### 実行時テスト

実行時テストでは、Orbs を含むアクティブなビルドを実行します。 ビルド内のジョブは、設定ファイルの一部である Orbs、またはビルドの開始時にパブリッシュされた Orbs にのみ依存するため、このテストを実行するには特別な計画が必要です。

1 つの選択肢として、CircleCI CLI を使用し、ビルド内の Machine Executor 上で `circleci local execute` を使用してローカル ビルドを実行し、ビルドのジョブ内でジョブを実行する方法があります。 こうすると、ビルド出力を `stdout` に表示してアサーションすることができます。 ただし、ローカル ビルドにはワークフローをサポートしないなどの注意点があるため、この方法を利用できないことがあります。 この方法は、Orb エレメントを使用するビルドの実際の実行出力をテストする必要がある場合にもたいへん便利です。

実行時テストは他にも、Orb エンティティをジョブの設定で直接使用するなどの方法で実行できます。

オプションとして、実行してテストするジョブの事後ステップ、または実行してテストするコマンドの後続ステップを使用してチェックすることも可能です。 これらのステップでは、ファイル システムの状態などをチェックできますが、ジョブとコマンドの出力についてはアサーションできません。

**メモ:** CircleCI では、この制限をなくせるように努めております。Orbs をテストする理想的な方法についてご意見がございましたらぜひお知らせください。

また、Orb ソースを `config.yml` として使用するという方法もあります。 Orb でジョブのテストとパブリッシュを定義する場合、それらのジョブは Orb で定義されているすべての項目にアクセスできます。 ただし、この方法では、無関係なジョブやコマンドを Orb に追加しなければならない場合があり、Orb がテストとパブリッシュに必要なすべての Orb に依存するという欠点もあります。

さらに別の方法として、ビルドの実行時に dev バージョンの Orb をパブリッシュしてから、その dev バージョンを利用する設定ファイルを使用するブランチに自動的にプッシュするという方法もあります。 この新しいビルドでテストを実行できます。 この方法の欠点は、実際のテストを行うビルドがそのコミットに直接関連付けられていないことと、Orb を変更するたびに複数のジョブを処理する必要があることです。

### インテグレーション テスト

Orbs と外部サービスとのやり取りについてテストしたい場合は、 状況に応じて以下のような方法でテストできます。

- やり取りするサービスのドライラン機能を Orb でサポートし、そのモードをテストで使用します。
- 適切に設定されたテスト アカウントを使用して実際にサービスとやり取りし、パブリッシュされた dev バージョンの Orb を使用してこのテストを実行するリポジトリを別途用意します。
- ジョブの別のコンテナでローカル サービスをスピンアップします。

## Orb テストのベスト プラクティス

Orbs テストにおける最大の問題は、Orb ソース コードを含むリポジトリに新しいコミットを単純にプッシュできないという点にあります。 これは、Orbs が展開後の `config.yml` にビルドの開始時に補間され、そのコミットに含まれる Orb には最新の変更が反映されていないことが原因です。

Orb と CircleCI プラットフォームの互換性が維持されるように Orbs をテストするには、インライン Orbs や外部リポジトリを使用するなど、いくつかの方法があります。 また、CircleCI では、Orbs テストの新しい方法の開発を進めています。

## 手法

### ローカルでの Orbs のテスト

Orbs をローカルでテストするなら、インライン Orb を作成すると簡単です。 インライン Orb を作成することで、アクティブな開発中でも Orb をテストできます。 インライン Orbs は、Orb の開発時や、長い設定に含まれるジョブやコマンドの名前空間を作成するときに使用すると便利です。

インライン Orb の作成方法の詳細については、CircleCI ドキュメントの「[Orbs の作成]({{site.baseurl}}/2.0/creating-orbs/)」ページを参照してください。

### Orbs テストの自動化

Orbs テストは、複数のレベルから選択して実行できます。 テストのレベルは、Orb の複雑さやユーザーの範囲を考慮して決定します。

どのレベルのテストでも、CircleCI CLI を使用して、Orb をローカルでバリデーションするか、ビルド内でテストを自動化することをお勧めします。 CLI のインストール手順については、CLI に関するドキュメントを参照してください。

高度なテストを実行する場合は、Bats などのシェル単体テスト フレームワークも使用できます。

## 関連項目

- [Orb のコンセプト]({{site.baseurl}}/2.0/using-orbs/): CircleCI Orbs の基本的な概念
- [Orb のパブリッシュ]({{site.baseurl}}/2.0/creating-orbs/): ワークフローやジョブに使用する Orb のパブリッシュ プロセス
- [Orbs リファレンス ガイド]({{site.baseurl}}/2.0/reusing-config/): 再利用可能な Orbs、コマンド、パラメーター、および Executors の例
- Refer to [Configuration Cookbook]({{site.baseurl}}/2.0/configuration-cookbook/) for more detailed information about how you can use CircleCI orb recipes in your configurations.