---
title: "複数の Git アカウントを使い分ける方法"
emoji: "👥"
type: "tech"
topics: ["Git"]
published: true
---

プライベート用と仕事用で Git のアカウントを使い分ける必要があったので、調べて設定したものをまとめてみました。

:::message
[GitHub は複数の無料アカウントを取得することを禁止している](https://docs.github.com/ja/github/site-policy/github-terms-of-service#3-account-requirements)ので注意してください。

> 1 人の個人または 1 つの法人が複数の無料「アカウント」を保持することはできません

:::

## モチベーション

- 複数のアカウントを所持していた
  - プライベートで使う GitHub 用のアカウント
  - 仕事で使う GitHub Enterprise Server 用のアカウント
- 複数のアカウントを簡単に切り替えたい
  - デフォルトの状態ではプライベート用アカウントを使いたい
  - 特定のディレクトリ以下では仕事用アカウントを使いたい
  - `git config` の切り替えを手動でやるのは面倒

## SSH キーを複数用意

プライベート用と仕事用にそれぞれ SSH キーを用意します。SSH キーの作り方は[GitHub Docs の新しい SSH キーを生成して ssh-agent に追加する方法](https://docs.github.com/ja/github/authenticating-to-github/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#adding-your-ssh-key-to-the-ssh-agent)を参照してください。

ファイル名は任意ですが、間違えないように区別しやすい名前にしておきましょう。

私は以下のようなファイル名にしています。

```plain
id_ed25519.private     # プライベート用秘密鍵
id_ed25519.private.pub # プライベート用公開鍵
id_ed25519.work        # 仕事用秘密鍵
id_ed25519.work.pub    # 仕事用公開鍵
```

それぞれ GitHub と GitHub Enterprise Server に公開鍵を登録しておきます。取り違えないように気をつけましょう。

## `~/.ssh/config` の設定

以下のように設定しておきます。

```plain:~/.ssh/config
# GitHub 用設定
Host github.com
  HostName ssh.github.com
  IdentityFile ~/.ssh/id_ed25519.private
  AddKeysToAgent yes
  UseKeychain yes
  TCPKeepAlive yes

# GitHub Enterprise Server 用設定
Host git.example.com # ホスティングしている場所に変更
  IdentityFile ~/.ssh/id_ed25519.work
  AddKeysToAgent yes
  UseKeychain yes
  TCPKeepAlive yes
```

GitHub Enterprise Server 用設定の `git.example.com` はホスティングしている場所に変更してください。

### SSH 接続確認

まずは GitHub への接続確認をします。

```shell
$ ssh -T git@github.com
Hi <プライベート用の名前>! You've successfully authenticated, but GitHub does not provide shell access.
```

GitHub に設定している名前が表示されれば問題ありません。

続いて GitHub Enterprise Server への接続確認をします。こちらも投げ先はホスティングしているものに変更して確認してください。

```shell
$ ssh -T git@git.example.com
Hi <仕事用の名前>! You've successfully authenticated, but GitHub does not provide shell access.
```

GitHub Enterprise Server に設定している名前が表示されれば問題ありません。

## `.gitconfig` の設定

[Conditional includes](https://git-scm.com/docs/git-config#_conditional_includes) を使うことで、特定の条件によって別の設定ファイルを追加で読み込むことができます。

今回は以下の要件で設定しました。

- デフォルトでプライベート用アカウントのコミッター情報を使用する
- `~/workspace/` 以下を仕事用の作業ディレクトリとする
  - このディレクトリ以下では仕事用アカウントのコミッター情報に切り替わる

まずは仕事用の設定ファイルを以下のように作成します。ファイル名は任意ですが、`.gitconfig.work` などの分かりやすい名前にしておきましょう。

```git:~/.gitconfig.work
[user]
  email = work@example.com
  name = <仕事用の名前>
```

続いて `~/.gitconfig` にプライベート用の設定と先ほど作成した仕事用の設定ファイルの条件付き読み込みを設定します。

```git:~/.gitconfig
[user]
  email = private@example.com
  name = <プライベート用の名前>
[includeIf "gitdir:~/workspace/**"]
  path = .gitconfig.work
```

### 設定が反映されているか確認

プライベート用の設定の確認は簡単で、`~/workspace/**` 以外の任意の場所で設定を確認します。

```shell
$ cd ~
$ git config --get user.name
<プライベート用の名前>
```

仕事用の設定の確認は一手間必要です。`~/workspace/` 以下の Git 管理をしているディレクトリ内で確認する必要があります。

適当なディレクトリに `git init` して確認してみましょう。

```shell
$ cd ~/workspace
$ mkdir test && cd test
$ git init
$ git config --get user.name
<仕事用の名前>
```

## 雑感

いつも `git config` で設定を切り替えていて面倒だったのがだいぶ快適になりました。
