---
title: "[GitHub Actions] 定期実行が発火しなかった際に確認したこと"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Github Actions"]
published: true
---

## はじめに

- GitHub Actionsの定期実行を利用した際に、予定通りの時間に発火しませんでした。状態を確認した経緯と結果をまとめています。
- 間違い等あれば、指摘いただけると助かります。

## 結論

- 結果として、定期実行は予定時刻より2～4時間遅れて発火した。
- 利用予定のシステムでは、数時間の実行遅延は許容できる設計に変更することにした。


## 動作環境

- Windows11
- Visual Studio Code
- GitHub

## 1\. GitHub Actionsとは

「GitHub Actions」とは、GitHub公式が提供している便利機能です。

GitHub Actionsでは、リポジトリに対するpushやPull Requestなどのイベントをトリガーとして、専用ファイルに定義しておいた処理（ワークフロー/workflow）を自動で実行します。

GitHub Actionsの利点の一つが、GitHubが提供する「ランナー」（GitHubホストランナー）を利用できることです。
ランナーとは、OSや各種ツールがあらかじめ用意された一時的な実行環境です。
GitHub Actionsを利用することで、ユーザーは自前の実行環境を用意することなく、処理を実行できます。

### 1\.1 GitHub Actionsの利用方法

GitHub Actionsを利用する場合は、実行したいGit管理のリポジトリに「.github/workflows」ディレクトリを作成し、YAML形式のワークフローファイルを配置します。

.github/workflowsのディレクトリ名は固定であるため、ディレクトリ名が異なるとGitHubにワークフローとして認識されません。
ワークフローファイルは、任意の名前を指定できます。

ワークフローファイルをコミットし、リモートにpushすると、設定したイベント発生時にワークフローが実行されます。

```Plain
.github
  └─　workflows
     　└─ workflowfile.yml
```

### 1\.2 ワークフローファイルの記述方法

ワークフローファイルは、YAML形式でワークフローの内容を指定できます。
代表的な項目は、以下の通りです。

| 項目    | 説明                                                           |
| ------- | -------------------------------------------------------------- |
| name    | ワークフローの名前。任意の名前を指定できる                     |
| on      | 実行タイミング                                                 |
| jobs    | 処理の単位。ジョブは通常それぞれ独立したランナー上で実行される |
| runs-on | 処理を実行するランナー                                         |
| run     | 実行するコマンド                                               |

```yml
name: GitHub Actions test #ワークフローの名前

on: #実行タイミング
  push:  # push時に実行
  workflow_dispatch: #手動実行

jobs: #処理内容
  test:
    runs-on: ubuntu-latest # GitHubが提供する最新のUbuntuランナーを指定
    steps:
      - name: test  # stepの名前
        run: echo "Hello test" #実行するコマンド
      - name: test2
        run: echo "Hello test2"
```

他にも「Action」と呼ばれる、GitHubやコミュニティが公開している再利用可能な処理を利用できます。

### 1\.3 GitHub Actionsの利用料金

publicリポジトリにおいては、GitHub Actionsは無料で利用できます。

privateリポジトリでは、利用しているGitHubプランによって月単位の無料利用枠が異なります。

無料枠を超えて利用した場合は、設定した支払い方法に従って課金されます。
アカウントに有効な支払方法が登録されていない場合、新たなジョブは実行されない状態になります。

privateリポジトリにおける、無料利用枠は以下の通りです。

| プラン           | ストレージ | 分 (月あたり) | キャッシュ ストレージ (リポジトリごと) | カスタム イメージ ストレージ |
| ---------------- | ---------- | ------------- | -------------------------------------- | ---------------------------- |
| Free             | 500 MB     | 2,000         | 10 GB                                  | 適用なし                     |
| Pro              | 1 GB       | 3,000         | 10 GB                                  | 適用なし                     |
| Free 組織向け    | 500 MB     | 2,000         | 10 GB                                  | 適用なし                     |
| Team             | 2 GB       | 3,000         | 10 GB                                  | 75 GB                        |
| Enterprise Cloud | 50 GB      | 50,000        | 10 GB                                  | 150 GB                       |


## 2\. 定期実行

定期的に自動で処理を実行するシステムを開発しており、定期実行ツールとしてGitHub Actionsを候補に入れていました。

事前にGitHub Actionsの定期実行は時間通りに実行されないという情報は聞いていましたが、実行時間のずれは許容できるシステムであること、無料で利用できること、管理コストが低いことから、候補としていました。

まずは、ワークフローファイルを作成して、定期実行がどれほど遅延するのか検証しました。

実行したのは、以下のワークフローファイルです。

```yml
name: cron Test

on:
  workflow_dispatch:

  push :

  schedule: # 定期実行（cron形式・UTCで指定）
    - cron: "0,30 * * * *"

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Set up Python
        uses: actions/setup-python@v6
        with:
          python-version: "3.13"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run Python
        run: python src/main.py
```

コードをチェックアウトし、Pythonをセットアップして依存ライブラリをインストールした後、main.pyを実行する構成です。

システムの実行は1日1回の予定ですが、今回は検証用として、30分に一回の定期実行を指定します。
ワークフローファイルが正常に認識されているかの確認のため、イベントトリガーとして、「push時」「workflow_dispatch（手動実行）」も指定しています。

main.pyファイルは以下の通りです。

```Python
def main():
    print("Hello, world!")

if __name__ == "__main__":
    main()
```

平日の午前9時に、publicリポジトリにpushし、デフォルトブランチであるmainブランチにマージしました。

GitHub画面を確認すると、リポジトリのActionsタブに、ワークフローが表示され、「push時」の処理が実行されていました。
続いて、「Run workflow」ボタンから手動実行を試みます。
「push時」と「手動実行」は、エラーなく処理が実行できました。

しかし、実行予定時間から1時間経過しても、定期実行は発火しません。

## 3\. 定期実行の状態の確認

GitHub Actionsの定期実行は、遅延が発生する可能性があると公式で発表されています。

>GitHub Actionsワークフローの実行負荷が高い時間帯には、イベントscheduleの実行が遅れる場合があります。負荷が高い時間帯には、毎時開始時が含まれます。負荷が十分に高い場合、キューに登録されているジョブの一部が破棄される可能性があります。遅延の可能性を減らすには、ワークフローの実行時間を別の時間帯に設定してください。

そのため、実行が遅れることは想定内でした。
ここから、環境の再検証、状況の確認、私の環境下での遅延時間の検証を始めました。

### 3\.1 環境の再検証

まず、GitHub Actionsで定期実行に必要な設定に不備がないかを確認しました。

- 1. リポジトリでGitHub Actionsが有効になっているか？
  → リポジトリの「Settings」タブ、左メニューの「Actions」→「General」を開き、Actions permissions（Actionsの権限）の設定を確認します。「Allow all actions and reusable workflows」もしくは「Allow OWNER, and select non-OWNER, actions and reusable workflows」となっていれば、GitHub Actionsは有効です。

- 2. ディレクトリ名が「.github/workflows」一致しているか？
  → ディレクトリ名が1文字でも異なるとGitHubはワークフローを認識しません。「workflows」の末尾の「s」を書き忘れないよう注意してください。

- 3. ワークフローファイルが、デフォルトブランチに存在するか？
  → scheduleイベントは、デフォルトブランチにワークフローファイルが存在する場合のみ発生します。デフォルトブランチでない場合は、デフォルトブランチにマージ後再検証します。

- 4. cronの指定時刻を日本時間（JST）でなく、協定世界時（UTC）で記載しているか？
  → GitHub ActionsのcronはUTCで指定します。JSTはUTCより9時間進んでいるため、日本時間から9時間引いた時刻を指定します。

- 5. YAML構文・ワークフローの記述が正しいか？
  → 構文エラーがあるとActionsが実行されません。

Push時と手動実行は成功しているため、設定1・2・5については問題ないと判断しました。
また、デフォルトブランチへマージ済みであることから、設定3も満たしていると考えます。
設定4も、分単位での指定を利用しているため、実行遅延の原因ではないと判断しました。

### 3\.2 状況の確認

定期実行の遅延が設定の問題である可能性が低くなったため、次に「gh」を利用してGitHub Actionsの実行状況を確認します。

「gh」はGitHubをコマンドラインから操作するための公式CLIツールです。
リポジトリの操作、Issueの作成、Pull Requestの作成、GitHubの認証など、GitHub全体の操作をCLIで行えます。
この「gh」を利用して、GitHub Actionsの実行状況を確認していきます。

インストールする場合は、以下のコマンドを実行してください。

```Bash
# Windows版（wingetを使用）
winget install --id GitHub.cli
```
```Bash
# Mac版（homebrewを使用）
brew install gh
```

インストール後は、GitHubにログインして、認証を行ってください。
```Bash
gh auth login
```

ログイン状態は以下のコマンドで確認できます。
```Bash
gh auth status
```

### 3\.3 ghによるGitHub Actionsの実行状態の確認

まず、リポジトリのワークフロー一覧を確認します。

```Bash
gh workflow list
```
![gh workflow listの結果](/images/workflow-list.png)

リポジトリ内の、ワークフローの一覧が表示されます。
「STATE」が「active」であれば、GitHubがワークフローとして認識していることを示します。

しかし、プログラム中のバグや、外部APIの接続失敗など実行時のエラーが起きている場合も、「active」と表示されます。
「STATE」はあくまで、ワークフローが正常に認識されているかを表す指標であり、ワークフローが正常に実行できているかの指標ではありません。

次に、`gh run list`コマンドを実行してください。

```Bash
gh run list
```

![1時間後のgh run listの結果](/images/run-list-1h.png)

`gh run list`コマンドは、実際に実行されたワークフローの一覧を表示します。
EVENT列には、ワークフローを開始したイベント（push、schedule、workflow_dispatchなど）が表示されるため、どのイベントが実行のきっかけになったかを確認できます。

STATUS列には、成功・失敗・実行中などの実行状態が表示されます。
複数回実行されるワークフローの場合は、`gh run list`コマンドを確認することで、実行毎の成否、トリガーとなったイベントを把握できます。

スクショを確認しますと、pushから1時間以上経過していますが、scheduleによる実行が存在しません。
この結果から、少なくとも実行履歴には、schedule をきっかけとしたワークフローはまだ作成されていないことが分かります。

そのまま、定期的に`gh run list`コマンドを実行して、ワークフローの状況を確認していきます。

pushから3時間後、`gh run list`コマンドの結果が以下の通りです。

![3時間後のgh run listの結果](/images/run-list-3h.png)

予定時刻から2時間半以上経過して、ようやく最初の定期実行が発火しました。

pushから8時間後、`gh run list`コマンドの結果が以下の通りです。

![8時間後のgh run listの結果](/images/run-list-8h.png)

30分に1回の定期実行を実験しましたが、8時間経って、発火したのは2回という結果でした。
最初の発火までに3時間以上かかり、その後の実行も2時間ほど間隔があいていることがわかります。


## まとめ

今回、システム開発の候補に挙がっていたGitHub Actionsの定期実行が、私の環境でどのくらいの遅延があるのかを検証しました。
利用率の高い平日昼間に実施し、毎時00分・30分は利用者が多い可能性がある時間帯を指定というやや厳しめの実証実験でしたが、初回は約3時間遅れ、その後も約2時間間隔で実行されたため、概ね2〜4時間程度の遅延が見られました。

開発予定のシステムは、もともと数時間の遅延は許容できる仕様ではありますが、今回の実証実験の結果を踏まえて、4時間程度の遅延を考慮した設計を行おうと思います。

今回の検証は平日昼間・公開リポジトリ・無料プランで実施した結果であり、時間帯やGitHub側の負荷状況によって遅延時間は変動する可能性があります。

-----

記事は以上です。
最後までお読みいただき、ありがとうございました。

## 参考情報一覧

この記事は以下の情報を参考にして執筆しました。


- [GitHub Actions入門](https://zenn.dev/praha/articles/9e561bdaac1d23) (最終更新 2024-05-12) (参照 2026-07-16)
- [【入門】GitHub Actionsとは？概要やメリット、使用例まとめ](https://www.kagoya.jp/howto/it-glossary/develop/githubactions/) (最終更新 2023-08-08) (参照 2026-07-16)
- [github actionsのスケジュール実行は時間通りには実行されない](https://qiita.com/h16kazu_dev/items/dc432ccf6ea36bca1279) (最終更新 2026-02-25) (参照 2026-07-16)
- [GitHub Actionsについて](https://docs.github.com/ja/actions/get-started/understand-github-actions) (参照 2026-07-16)
- [ワークフローをトリガーするイベント](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows) (参照 2026-07-16)
- [GitHub CLI のクイック スタート](https://docs.github.com/ja/github-cli/github-cli/quickstart) (参照 2026-07-16)