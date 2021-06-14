# Jetson Nano 開発者キット python 開発環境構築

## はじめに
今回，次回の2回で，Jetson Nano開発者キット（以下,Jetsonに省略）でpythonを使った開発を行うための環境構築に関する事柄の入門を行います．

以下を実現することを目標に，2回の講習は進んでいきます．

1. DockerをJetson Nano上で動作させ，コンテナを立ち上げる．
1. 作業PCで開いたVScodeからJetson Nano上のDockerコンテナにアクセスし，Jetson Nanoを直接操作することなく，ソースコードの編集，実行を行う環境を構築する． 


そもそもなぜこんなことをする必要があるかを説明します．
皆さんはDeep Learning Instituteの教材を通して，Jetson Nanoを動かしてきました．

その際はJetson Nano上に立ち上がったJupyter Labにアクセスし，ノートブックをブラウザで立ち上げ，ノートブック上でコードの編集，実行を行っていました．

このノートブックはインタラクティブに入力，実行を行うことができますが，これはアプリ開発には不向きです． ほとんど全てのアプリケーションの本番環境ではpythonスクリプト(`***.py`)が使われていることでしょう．

ゆえにpythonスクリプト書き,コマンドラインからpythonを実行する環境を整えて，この環境を中心としてアプリ開発を行うことが推奨されます．

### 参考
 [iPythonのドキュメント](https://ipython.readthedocs.io/en/stable/)には以下のように述べられており，この記述からもあくまで Jupyterがソースコードと実行結果を「人間にとって」見やすくしてくれるに過ぎず,ノートブックの利点がそれに尽きることを物語っています．

> The IPython Notebook is now known as the Jupyter Notebook. It is an interactive computational environment, in which you can combine code execution, rich text, mathematics, plots and rich media. For more details on the Jupyter Notebook, please see the Jupyter website.

<br>
<br>
<br>
<br>

## 今日すること
今回は，

1. Jetsonに作業用PCからリモートアクセスできるようにする．
2. 作業用PCのエディタからJetsonの中のソースコードを編集，実行する．
2. pythonの仮想環境を理解する．

この３つを手順を１つ１つ追い，適宜説明をはさみながら環境を構築していきます．
Dockerについては次回扱います．その前段階の準備になります．

<br>
<br>
<br>

## Jetson NanoにSSHで接続しよう

### 概要
まずは，JetsonにSSHを使ってリモートアクセスして，普段皆さんが使っているPCからJetsonのCLIによる操作を可能にします．

### SSHとは
Secure Shellの略．ネットワークに接続されたデバイスを安全に遠隔操作する手段です．
通常はCLIで操作します．SSHにより，適切な設定を行えば，公開鍵暗号方式と呼ばれる通信暗号方式で通信を行うことが可能です．

#### 公開鍵，秘密鍵とは何か
A,Bが通信する際，

１．Aが秘密鍵から公開鍵を作成しBに渡す
２．Bが、その公開鍵を使って通信内容を暗号化する
３．暗号化された文書をAが受け取る
４．Aが秘密鍵を用いて復号し中身を確認する

<br>
<br>

それでは作業用PC側で秘密鍵と公開鍵を発行し，Jetson側に公開鍵を渡すことでSSHによる通信を行います．

<br>


### クライアント側（windowsの方）
まずは
powershellを起動，以下のコマンドが通るかどうか確認．
```
PS> ssh
```
##### 通らない人は以下を行ってください．
コルタナで`オプション機能の管理`を検索，開く．
機能の追加から，`OpenSSHクライアント`をインストール．
powershellを再起動して`ssh`コマンドが通るか確認してください．


#### 秘密鍵，公開鍵を作成
```
PS> ssh-keygen -t rsa -b 4096
PS> cd ~/.ssh/
```
`$ ssh-keygen -t rsa -b 4096` 
まず，クライアント側で公開鍵と秘密鍵を作成します．`-t`オプションで暗号化方式を決めます．`-b`で鍵のビット数を指定できます．

これで，ホームディレクトリ直下に`.ssh/`ができ，この中に秘密鍵ファイル`id_rsa`，公開鍵ファイル`id_rsa.pub`が作成されます．

#### アクセス権限の変更

id_rsaのファイル名をid_jetsonに変更．
続いて以下を実行してパーミッションを変更．
```
PS> cacls id_jetson /g [user名]:F
PS> cacls ~/.ssh /g [user名]:F
```
各ファイルやディレクトリの権限を設定します．（詳しくはここでは説明しません．）

ユーザ名の確認は以下で行えます．
```
PS> $userinfo = [System.Security.Principal.WindowsIdentity]::GetCurrent()
PS> $userinfo.Name
```

### クライアント側（macOS, linuxの方）
最初からsshは使えるはずです.


#### 秘密鍵，公開鍵を作成

```
$ ssh-keygen -t rsa -b 4096
$ cd ~/.ssh/
$ cat id_rsa >> id_jetson
```

`$ ssh-keygen -t rsa -b 4096` 
まず，クライアント側で公開鍵と秘密鍵を作成します．`-t`オプションで暗号化方式を決めます．`-b`で鍵のビット数を指定できます．

これで，ホームディレクトリ直下に`.ssh/`ができ，この中に秘密鍵ファイル`id_rsa`，公開鍵ファイル`id_rsa.pub`が作成されます．

`cat`で`id_rsa`を`id_jetson`にコピー．

#### アクセス権限の変更
```

$ chmod 600 id_jetson
$ chmod 700 ~/.ssh
```

各ファイルやディレクトリの権限を設定します．（詳しくはここでは説明しません．）


<br>
<br>

### ホスト側

#### 公開鍵をホストへ渡す
`id_rsa.pub`をUSBメモリやgoogle dirve等を使ってクライアント側の`~/.ssh`に設置してください．
(パスワード認証方式を使った方法もあります．)

#### 公開鍵を追加

公開鍵の名前を`authorized_keys`に変更し，権限を変更してください．

```
$ cat id_rsa.pub >> authorized_keys
$ chmod 600 authorized_keys
$ chmod 700 ~/.ssh
``` 

<br>
<br>

### クライアント側　コマンドでsshアクセス
以下のコマンドでアクセス可能です．秘密鍵の場所を指定し，ユーザIDとIPアドレスを指定してください．

IPアドレスは,ヘッドレスデバイスモードの場合,`192.168.55.1`が割り当てられています．

```
$ ssh –i ~/.ssh/id_jetson [ログインユーザID]@[IPアドレス]
```

これで，sshによりJetson Nanoにアクセスし，作業PCのpowershellやterminal上で，Jetson NanoをCLIで操作可能になりました．

<br>
<br>

### .ssh/config
秘密鍵の場所，ユーザー名，IPアドレスを毎回手打ちするのは面倒です．そんな時に便利なのが，`.ssh/config`になります．
クライアント側の`~/.ssh/`に`config`というファイル名でテキストファイルを作成してください．テキストエディタでこれを開き，以下のように編集しましょう．

```
Host jetson_usb_mode
    HostName IPアドレス
    User ユーザー名
    IdentityFile ~/.ssh/id_jetson
```
以上の設定を終えると，Hostに示した名前で接続が可能になります．


```
$ ssh jetson_usb_mode
```

### 演習

ここまでできた人は，LANケーブルを用意して，デバイスモードではなく，LANケーブルによるSSH接続環境を構築してみましょう．

<br>
<br>
<br>
<br>


## VScodeからJetson Nanoに接続する．
### VScodeとは
Visual Studio CodeはMicrosoftが開発しているWindows、Linux、macOS用のソースコードエディタです．コード編集の他，git，githubコントロールやデバッガ，インテリセンスなどなど他にもここでは語り尽くせないほどたくさんの便利機能，拡張機能を備えたエディタです．あくまでエディタなのでVScode自体にプログラムをコンパイルして実行する能力はありませんが，設定次第でそれらの操作が全てVScodeから行えるようになります．
### VScodeをインストールしよう．
こちら無料です．

https://code.visualstudio.com/download

このページからそれぞれ自分の作業用PCの環境にあったVScodeをダウンロードしてください．

### 拡張機能：Remote-Developmentを取得．
VScodeはそのまま使うとローカルに保存されたディレクトリを開き，そこでソースコードを編集することが可能です．

さらに，拡張機能である`Remote-Development`を導入することで，
リモートで接続した先のマシン内のソースコードをあたかもローカルに存在しているファイルかのように取り扱うことができるようになります．

`Remote-Development`は
`Remote-WSL`，`Remote-SSH`，`Remote-Container`
の3つからなる拡張機能で，今回はこのうちの`Remote-SSH`を使うことで，Jetsonにアクセスしたいと思います．

### Remote-SSH
参考資料:https://code.visualstudio.com/docs/remote/ssh 
VScodeを開いてください．

サイドバーのアイコンで，四角形が４つ並んでいるアイコンを選択してください．
ここから様々な拡張機能の管理を行うことができます．

`Remote-Development`拡張機能をインストールしたら，
VScodeのサイドバーに，ディスプレイにマークがついているアイコンが増えているはずです．

こちらを選択し，一番上のプルダウンからSSH Targetを選択します．

続いて一番下のOpen a remote windowをクリックし，

Connect to SSHを押してください．

すると，`.ssh/config`に書き込んだ`Host name`が表示されているはずです．

こちらをクリックしてSSH接続が完了するのを待ちます．

完了すると，別の新しいウインドウでJetson内に接続した状態のVScodeのワークスペースが立ち上がるはずです．

### VScode環境を試してみよう．
一旦ターミナルを開いてください．（ショートカットはshift+ctr+@）

ここでこれから編集する適当なディレクトリを作成しましょう．

作成したら，VScodeウインドウ左部分にあるEXPLORERから今作ったディレクトリを開いて見てください．新しくtest.pyを作成します．

`python`拡張機能を入れますか？と出ますが，このまま開発を行うわけではないので，今はインストールしません.

```
print("hello, world.")
```
と`test.py`を編集後，

VScodeのterminalにて以下のコマンドで実行してください．
```
$ python3 test.py
```

これで，

1. Jetsonに作業用PCからリモートアクセスできるようにする．
2. 作業用PCのエディタからJetsonの中のソースコードを編集，実行する．

この２つが実現できました．

<br>
<br>
<br>
<br>

## python仮想環境を作成

Python アプリケーションはよく標準ライブラリ以外のパッケージやモジュールを利用します。

プロジェクトによって必要になるpythonパッケージは異なります．時には異なるバージョンのパッケージを同時に異なるアプリケーションで動かしたいこともあります．

パッケージによる互換性や依存性に左右されることなく，各プロジェクトで独立に実行環境を用意するため，プロジェクトを始める際は「仮想環境」を導入する必要があります．

仮想環境とは、特定のバージョンの Python と幾つかの追加パッケージを含んだ Python インストールを構成するディレクトリです。
これにより，プロジェクトごとに独立したpythonの実行環境を整備することが可能です．

独立したPythonの実行環境は、他の環境に影響を与えずにPythonのバージョンを用途によって切り替えたり、パッケージをインストールしたりできます。

今回は最終的に皆さんにはDockerを使えるようになっていただくため，jetsonのホストマシン環境上でダイレクトにpythonパッケージ管理ソフトを動かすことはありません．
しかしながら，仮想環境を作成できるモジュールを一度も触らずにいきなりDockerにトライするのも大変です．
そこで今回は皆さんが普段使っているPCにanaconda環境を構築していただき，そこで実際にanacondaを触っていただき，「仮想環境」という概念を知っていただこうと思います．



### anacondaのインストール

事前にダウンロードしていただいた，それぞれの環境のAnaconda installerを使ってAnacondaをインストールしてください．

### Windowsの方
anaconda promptを起動しましょう．

### MacOSの方
インストールが完了したら，`terminal`を起動してください．

以下コマンドを実行して現在使用しているシェル名を確認しましょう．(OSがcatalina以降であればデフォルトは`zsh`のはずです．)
```
$ echo "$SHELL"
```
続いて

```
$ /opt/anaconda3/bin/conda init シェル名
```
を実行してください．
これでターミナルを再起動すると
```
(base) ***@*** ~ $
```
以上のようにcondaのbase環境に入っている状態になります．


### Anacondaで仮想環境を作成

conda環境が有効な状態では，現在いる環境のpythonを使うことができます．

以下のコマンドでpythonのインタラクティブシェルモードを起動します．
```
(base) $ python
```
すると現在起動した`python`のバージョンなどの情報を得ることができます．
Anacondaのpythonが動いていることが確認できると思います．

```
Python 3.8.5 (default, Sep  4 2020, 02:22:02) 
[Clang 10.0.0 ] :: Anaconda, Inc. on darwin
Type "help", "copyright", "credits" or "license" for more information.
```

インタラクティブシェルモードは以下のコマンドで終了することができます．

```
>>> exit()
```

#### パッケージリストの閲覧
まずは，最初に起動したベース環境にどんなパッケージがインストールされているのかみてみます．

```
(base) $ conda list
```

パッケージの名前，バージョン等の一覧が表示されたかと思います．
Anacondaのbase環境には科学計算においてよく使われるパッケージがあらかじめインストールされています．

#### conda環境の作成
それでは，`base`とは別の新しい環境を作成してみましょう．
例として，異なるpythonのバージョンの環境を用意してみましょう．
現在の`base`環境のpythonバージョンは,以下のコマンドで確認できます．

```
(base) $ python --version
```
それでは，以下のコマンドでpython 3.6の環境を作成します．

```
(base) $ conda create --name py36 python=3.6
```
#### 作成済みconda環境の一覧 
作成した環境を確認しましょう．
以下コマンドで環境の一覧を確認できます．

```
(base) $ conda info -e
```

#### conda環境の切り替え
それでは環境を切り替えて,
先ほど作成した`py36`環境に入ります．
カッコの中の環境名が切り替わったこと，pythonのバージョンが切り替わったことを確認してください．

```
(base) $ conda activate py36
(py36) $ python --version
```

#### パッケージのインストール
それではこの環境に新しいパッケージを追加してみます．今回はtensorflowを追加します．
実行後出てくる大量のメッセージはパッケージ間の依存関係を解決してくれたりしている様子です．
```
(py36) $ conda install tensorflow=1.15
```

インストールが完了したら，インタラクティブシェルを起動してtensorflowをimportしてみてください．

```
(py36) $ python
>>> import tensorflow
>>> print(tensorflow.__ver__)
```

確認できたら再び`base`に環境を切り替えて，今度はこちらでインタラクティブシェルを起動してtensorflowをimportしてみてください．


`tensorflow`はこちらにはインストールされていないことを確認できましたか？

続いてpython3.7の環境を`py37`という名前で作成し，この環境に入ってから，tensorflow 2.0をインストールしてみてください．

それぞれの環境で，異なるpythonのバージョン，パッケージ，パッケージのバージョンを独立して扱うことができることがわかったかと思います．
これが仮想環境です！


#### その他
他にも主要なコマンドがあります．[公式ドキュメント](https://conda.io/projects/conda/en/latest/commands.html)もぜひ見ていただき，仮想環境を使ってpythonの実行環境を整える習慣を身につけていただければと思います．

### 注意
あくまで今回は，仮想環境の導入として`conda`を取り上げました．
他にも仮想環境を作る方法はいくつかあるので，そのうちの１つであることに留意してください．

jetson上でのcondaの使用に関しては動作検証ができていないため，私の方では保証できません．
pythonの具体的な仮想環境構築は次回のDocker入門にて説明します.来週こちらを身につけることができれば，マシン直上に
condaを導入する必要はなくなります．
