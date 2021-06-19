# Flaskを使ったwebアプリケーションの作成1
## 事前準備
- Jetson にインストールされたJetpack のバージョンが4.5.0以上であることを確認してください．

- Jetsonを起動する前にCSIカメラを接続してください．
接続したら起動して，JetsonにVScodeのRemote-SSHを使ってログインしましょう．
```
$ ls /dev/video0
```
以上のコマンドを使ってCSIカメラの接続を確認してください．


## 前置き
今回，次回で皆さんにJetsonを使って簡単なwebアプリケーションを作成していただきます．内容としては，Jetsonにwebサーバを立て，ブラウザからJetsonに接続したカメラで撮影した映像をストリーミングします．そしてその映像に対して物体検出アルゴリズムを適用し，物体検出が行われた状態の動画をストリーミングします．


## webの仕組み
皆さん，webブラウザ(google chrome，safari,Mozilla Firefox,Microsoft edgeなど)を使ってwebページを見ることは毎日のように行っているのではないでしょうか．
webに接続されたコンピューターはクライアント (client) とサーバ(server) と呼ばれます。
関係図は以下の通りです．
![](https://i.imgur.com/9Th7GlU.png)

### クライアント
一般的なwebユーザが使うインターネットに接続された端末のことを指します．例えばWi-Fi に接続されているコンピュータなどです．さらには，これらの端末で利用可能なwebにアクセスするソフトウェア(webブラウザ)のこともクライアントと呼びます．

### サーバ
Webページ，サイト，アプリを格納しているコンピュータを指します．クライアント端末がwebページにアクセスするとき，webページのコピーがサーバからクライアントにダウンロードされ、ユーザのwebブラウザに表示されるます．

### リクエストとレスポンス
webに接続されたクライアントとサーバは要求と応答のやりとりを繰り返すことで処理が進んでいます．
例えば，webページを閲覧することを想定してください．
まずブラウザのアドレスバーにURLを打ち込みます．これが「URLに紐づけられたページを見せて」という要求(リクエスト)になります．これに対してサーバは該当するHTMLファイルを渡します．
![](https://i.imgur.com/E6yUYEe.png)

## webアプリケーションとは

### 概要
インターネット（ウェブ）などのネットワークから利用するアプリケーションソフトウェアです。webアプリケーションはwebサーバ上で動作し、webブラウザで操作します。
みなさんが普段使っているものの例としては，「YouTube」などがそうです．

### ネイティブアプリとの比較
ネイティブアプリとは、手元の端末（スマートフォン・PCなど）にインストールして利用するアプリケーションソフトウェアをさします。webアプリではプログラム本体はwebサーバ内にあるのに対し、ネイティブアプリのプログラム本体が保存されるのは手元の端末内です。世の中で利用されているアプリケーションにはweb版とネイティブ版の両方がある場合があります．
例えば「slack」にはブラウザからログインして操作するものとスマホやPCにインストールして使うものの2種類があることを皆さんご存じかと思います．

## web開発に関する参考資料
こちらの資料にwebの仕組みからweb開発の方法，チュートリアルに至るまで，必要な情報がまとめられています．余力のある方はwebの仕組みを理解しながら今回のアプリを作成してみてください．

[MDN Web Docs](https://developer.mozilla.org/ja/)


## Flaskとは
pythonのwebアプリケーションのサーバサイドフレームワーク．
フレームワークとは，Webアプリケーションやシステムを開発するのに必要な機能が予め用意された骨組みのことです．pythonに限らず，他にも様々なフレームワークが存在します．(Ruby on Rails，Djangoなど)

> 最小限の機能、軽量・高速なフレームワーク。
Flask（フラスク）は軽量なWebフレームワークです．標準で提供されているパッケージが必要最小限であることから、「マイクロフレームワーク」と呼ばれるます。WebサーバとアプリケーションをつなぐインターフェースであるWerkzeugとテンプレートエンジンであるJinja2を基に作られている。




## flask環境の構築
Jetsonにアクセスしましょう．
### プロジェクト用ディレクトリを作成
```
$ mkdir ~/streaming_app
$ cd ~/streaming_app
```
#### 
Dockerコンテナ上に構築しましょう.

```
$ touch Dockerfile
```
Dockerfileを作成したら，以下の内容で書き込んでください．

```dockerfile=
FROM nvcr.io/nvidia/l4t-ml:r32.5.0-py3

RUN apt-get update \
    && apt-get upgrade -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && pip3 install -U pip \
    && pip3 install uwsgi==2.0.19.1 \
    && pip3 install flask==1.1.2

RUN apt-get update \
    && apt-get install nginx -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN python3 --version \
    && pip3 --version

WORKDIR /usr/app

CMD ["bash"]
```
上記Dockerfileについて必要最低限の説明をします．

ベースとするDockerイメージは`nvcr.io/nvidia/l4t-ml:r32.5.0-py3`です．
タグに書かれた，イメージが対応するJetpackのバージョンと使っているJetpackのバージョンが対応しているか確認しましょう．`r32.5.0-py3`タグはJetpack4.5.0以降で使用可能です．

8行，9行目ではuWSGIとFlaskをインストールしています．さらに12~15行目ではnginxをインストールしています．バージョンは変更しないでください． uWSGI，nginxについては後述します．

それではこのDockerfileをビルドします．

```
$ docker build -t flask_uwsgi_nginx:stream .
```
つづいてこのイメージからコンテナを作成します．以下の内容を`docker_run.sh`を作成して書き込みましょう．
```=
#! /bin/bash

docker run -it \
--runtime nvidia \
--network host \
-v ~/streaming_app:/usr/app \
--volume /tmp/argus_socket:/tmp/argus_socket \
--device /dev/video0 \
--name flask_app_con flask_uwsgi_nginx:stream \
```

ここで，7行目に書いたボリュームマウントはCSIカメラを使う際に必ず必要なものです．

以下のコマンドでコンテナを作成しましょう．
```
$ bash docker_run.sh
```
一旦コンテナを抜けて，起動しましょう．
```
# exit
$ docker start flask_app_con
```


新しくVSCodeのWindowを開き，そこからremote Containersで`flask_app_con`にアクセスして，`/usr/app`を開きましょう．
<!-- ## for CSIカメラ
```
# apt install protobuf-compiler libprotoc-dev libjpeg-dev cmake
# export path=$PATH:/usr/local/cuda/bin
# pip3 install -r requirements.txt
``` -->

## アプリケーションの作成
### ディレクトリを作成
まずは実際にflaskを使っていきましょう．
以下のように，これから作成したいプロジェクト用のディレクトリを作成し，作成したディレクトリに移動して，`app.py`を作成しましょう．
```
$ mkdir /usr/app/tutorial
$ cd ~/usr/app/tutorial
$ touch app.py
```

### Hello, World!

それでは，先ほど作成した`app.py`に以下のように書き込んでください．

```python:app.py
from flask import Flask

app = Flask(__name__)


@app.route("/")
def hello_world():
    return "Hello, World!"


app.run(port=8000, debug=True)

```
完了したら，以下のように実行しましょう.


```
$ python3 app.py
```
以下が表示されると，成功になります．
![](https://i.imgur.com/EEDR7kE.png)

しばらく待つと，VScodeに
![](https://i.imgur.com/6UZWpAX.png)
以下のような通知が来ますので，こちらで`Open in Browser`を選んでください．

すると，以下のようにBrowserで表示された文字列を見ることができます．

![](https://i.imgur.com/tswFBQj.png)


この出力には，以下の記述が見られますね．

```
Running on http://127.0.0.1:8000/ (Press CTRL+C to quit)
```


### Hello, World! 解説 

#### app.pyについて

まず，先ほど書いた`app.py`に書かれていることを説明します．

```python=
from flask import Flask

app = Flask(__name__)


@app.route("/")
def hello_world():
    return "Hello, World!"


app.run(port=8000, debug=True)

```
1行目でFlaskをインポートしています．
もう少し詳しくいうと，このクラスのインスタンスはWSGI（訳注: Pythonで標準化されている、WebアプリとWebサーバ間のインタフェース）アプリケーションになります。

3行目でappにFlaskクラスのインスタンスを作成しています．

続いて6~8行目の処理について説明します．
ここでは，関数の中に行いたい処理が記述されており，デコレータを使って「ルーティング」が行われています．


デコレータとは，（`@app.route("/")`：この部分）
すでにある関数に処理の追加や変更を行う為の機能です。関数の直前の行に`@`から始まって記述されます．

ルーティングとは，
クライアントからのリクエスト内容と、サーバーの処理内容を紐付ける作業のことを指します．

webでは基本的に，クライアント側で指定されたURLが引き金となり，そのURLで表されるリクエストと結び付けられた処理が呼び出されます．ほとんどの場合．その処理の結果がレスポンスとしてクライアント側のブラウザで開いているページに反映されたり，ページ遷移が起きたりなどします．

よって，ここでは，6行目の('/')でアドレスを指定して，それ以下の部分にそのアドレスと紐づけられた関数`hello_world`が記述されています．

最後の11行目の
```
app.run(port=8000, debug=True)
```
では，アプリケーションを開発用の仮サーバで起動します．
ポート番号を指定しています．

<br>
<br>

#### webページへのアクセス

先ほど，ここまで書いたコードを
```
$ python3 app.py
```
として実行しました．
すると，

![](https://i.imgur.com/EEDR7kE.png)
これが表示されました．

ここには，
```
Running on http://127.0.0.1:8000/ (Press CTRL+C to quit)
```
以上のことが書かれていますね．これは，開発用サーバに上記アドレスでアクセスできることを示しています．

`127.0.0.1`がIPアドレスで，`8000`がポート番号です．
IPアドレスとは，
ネットワーク上の情報機器を識別するために指定する番号です．

そしてポート番号とは，
コンピュータが通信に使用するプログラムを識別するための番号です。
プログラムごとにデータの送受信に使用するポート番号を割り当てます．
もし外部からそのプログラムとやりとりしたい場合は，そのポート番号を指定します．

今回は，8000番ポートにflaskの開発サーバとのやりとり出入り口が割り当てられているため，`8000`番を指定します．

このアドレスにブラウザでアクセスすると，
以下のページが出てくるはずです．


![](https://i.imgur.com/tswFBQj.png)

続いて，
VScodeのターミナルウィンドウで以下のことを行ってみてください．

まず，`PORTS`タブを選択して移動してください．

![](https://i.imgur.com/CaLYPYv.png)

次に8000番のポートに関する行にカーソルをホバーリングして表示されるバツ印をクリックしてください．

![](https://i.imgur.com/yR1D2hC.png)


それからブラウザで，
もう一度`http://127.0.0.1:8000/`にアクセスしてみてください．

![](https://i.imgur.com/jDso2nt.png)

このように，
「Hello, World!」と書かれたページにアクセスできなくなってしまいました．
これはどういうことなんでしょうか．

#### ポートフォワーディング
先ほど操作した`PORTS`タブは「ポートフォワーディング」の管理を行うことができます．

先ほどアクセスしようとしたアドレスの`http://127.0.0.1:8000/`を見ていきます．
IPアドレス`127.0.0.1`はローカル・ループバック・アドレスと呼ばれ、自分自身を指す特別なIPアドレスです．特別なアドレスのため，`localhost`という文字列も同様の意味を持ちます．


この場合は，Jetsonの上に立ち上げたコンテナ「`flask_con`」上でこのプログラムを実行しているため，`127.0.0.1`によって示されているのは「`flask_con`」コンテナです．

従って，先ほどの
```
Running on http://127.0.0.1:8000/ (Press CTRL+C to quit)
```
という出力は，「flask_conコンテナ上の」8000番ポートにアクセスするとwebページを見られる，という意味だったのです．
![](https://i.imgur.com/byb96ER.jpg)
イメージ



一方で，作業しているPCからこのIPアドレス`127.0.0.1`を参照すると，このアドレスは作業しているPCを指します．


そのため，特別な設定を行わずに，作業PCから`http://127.0.0.1:8000/`を参照すると，自分のPCの8000番ポートにアクセスすることになります．

今回，作業PCの8000番ポートには何も割り当てていませんから，このアドレスにアクセスした時，「このサイトにアクセスできません」と表示されたのです．


![](https://i.imgur.com/KUqEcua.jpg)
イメージ



しかし，以下のメッセージが表示された際．
![](https://i.imgur.com/6UZWpAX.png)

Open in Browserをしてページを表示することができました．

アドレスバーに書かれていたアドレスは，`http://127.0.0.1:8000/`でした．これは，「ポートフォワーディング」をVSCodeが自動で行ってくれたため，ページを開くことができたのです．

「ポートフォワーディング」とは，
ローカル(今回は作業用PC)のあるポート番号Aにアクセスすると，その通信をSSH接続先(今回はJetson上のflask_conコンテナ)のポート番号Bに転送してくれる機能です．
これにより，ブラウザで`127.0.0.1`（作業用PC）の`8000`番にアクセスしたのに，実際に通信を行っている相手は，`flask_con`のポート`8000`と通信していることになり，webページにアクセスできました．

![](https://i.imgur.com/B7IFsw1.jpg)
イメージ


改めてVScodeの`PORTS`を見てみましょう．
![](https://i.imgur.com/yJ6ECgD.png)

一番左の`PORT`に書かれた番号が転送先，つまり`flask_con`のポート番号です．
そして`Local Addrress`が転送先に通信が送られるローカルのアドレスになります．
今回は分かりやすいようにどちらも`8000`にしましたが，番号は一致していなくても構いません．



## 静的ファイルの表示
先ほどはルーティングされた関数の返り値の文字列がそのまま表示されるページでしたが，
webページはHTMLを使ってページを編集できます．

ここからはtutorialの中で操作しますので，一旦VScodeでフォルダを開きなおし，`tutorial`を開いた状態にしてください．`tutorial`内に`templates`ディレクトリを作成してください．そこに`index.html`を作成しましょう．

```
# mkdir /usr/app/tutorial/templates
# cd /usr/app/tutorial/templates
# touch index.html
```

以下のように`index.html`を編集してください．

```htmlembedded=
<!DOCTYPE html>
<html lang="ja">
<head>
    <title>Jetsonを使った開発</title>
</head>
<body>
<p>Flask</p>
</body>
</html>
```

### ルーティング
`app.py`を以下のように編集してください．
```python=
from flask import Flask, render_template

app = Flask(__name__)


@app.route("/")
def hello_world():
    return "Hello, World!"

@app.route('/home')
def index():
    return render_template('index.html')

app.run(port=8000, debug=True)
```
`render_template`はアプリケーションのディレクトリに存在する，`templates`ディレクトリに入っている指定したファイル名のhtmlファイルを出力してくれます．ここでは`/home`でルーティングしています．

```
# python3 app.py
```
して,`127.0.0.1:8000/home`に接続してください．

![](https://i.imgur.com/sCWj9KJ.png)

表示されました．

HTMLの書き方はここでは説明しません．先に紹介したMDN　Docsにわかりやすいドキュメントがありますし，他にも沢山ネットに情報があるのでそちらを参照してください．


## Streaming機能の実装
.
これから，webページにCSIカメラで撮影している動画をリアルタイムで表示するストリーミング機能を実装します．

新たに`camera.py`を`tutorial`直下に作成し，
```python=
import cv2

from utils import Singleton

gstreamer_pipeline = 'nvarguscamerasrc sensor-id=0 ! video/x-raw(memory:NVMM), width=640, height=480, format=(string)NV12, framerate=(fraction)30/1 ! nvvidconv ! video/x-raw, width=640, height=480, format=(string)BGRx ! videoconvert ! appsink'

class Camera(Singleton):
    def __init__(self):
        self.video = cv2.VideoCapture(gstreamer_pipeline, cv2.CAP_GSTREAMER)

    def __del__(self):
        self.video.release()

    def get_image(self):
        success, image = self.video.read()
        return success, image

```

上のクラスを使い，openCVを使ってカメラで撮影された映像を取得します．

また，`utils.py`を`tutorial`直下に作成し，以下を書いてください．
おまじないと思っていただいて構いません．
```python=
class Singleton(object):
    def __new__(cls, *args, **kargs):
        if not hasattr(cls, "_instance"):
            cls._instance = super(Singleton, cls).__new__(cls)
        return cls._instance
```


また，`templates`ディレクトリに，`stream.html`を作成し，以下のようなストリーミング用のページを記述します．
```htmlembedded=
<html>

<head>
	<title>Video Streaming Demonstration</title>
</head>

<body>
	<h1>Video Streaming Demonstration</h1>
	<img src="{{ url_for('video_feed') }}">
	<!-- jinja2のテンプレートの書き方です。/video_feedを呼び出しています。 -->
	<p><a href="/kill_camera">カメラを終了する</a></p>
	<p>切断する前に必ずカメラを終了してください．</p>
</body>

</html>
```

続いて，先程作ってあった`index.html`を以下のように編集しましょう．

```htmlmixed=
<!DOCTYPE html>
<html lang="ja">

<head>
	<title>Jetsonを使った開発</title>
</head>

<body>
	<p>Flask</p>
	<p><a href="/stream">ストリーミングを開始する．</a></p>
</body>

</html>
```
さらに`templates`の中に，`camera_killed.html`を作成してください．

```htmlmixed=
<html>

<head>
    <title>Video Streaming Demo</title>
</head>

<body>
    <h1>Cameraが終了しました．</h1>
    <p><a href="/stream">カメラを再起動する</a></p>

</body>

</html>
```
最後に，`app.py`を以下のように書き直します．
```python=
from flask import Flask, render_template, Response
import cv2

from camera import Camera

app = Flask(__name__)

def gen_video(camera):
    while True:
        success, image = camera.get_image()
        if not success:
           break

        ret, frame = cv2.imencode(".jpg", image)
        yield (
            b"--frame\r\n"
            b"Content-Type: image/jpeg\r\n\r\n" + frame.tobytes() + b"\r\n"
        )

@app.route("/")
def hello_world():
    return "Hello, World!"

@app.route('/home')
def index():
    return render_template('index.html')

@app.route("/stream")
def stream():
    return render_template("stream.html")

@app.route("/video_feed")
def video_feed():
    cam = Camera()
    return Response(gen_video(cam), mimetype="multipart/x-mixed-replace; boundary=frame")


@app.route("/kill_camera")
def kill_camera():
    cam = Camera()
    del cam
    return render_template("camera_killed.html")

if __name__ == "__main__":
    app.run(port=8000, debug=False, threaded=True)

 
```
それではここまでの内容を説明します．

## 解説
### 全体像
![](https://i.imgur.com/50z30ne.jpg)



### 各種htmlファイル
#### "ハイパーリンク"
```htmlembedded=
<p><a href="/stream">カメラを再起動する</a></p>
```
これでハイパーリンクをページ上に作成できます．`href`に，この文字列をクリックしたときに遷移したいページなどのアドレスを指定することで，アドレスバーにURLを手打ちせずとも，ページを切り替えることができます．

###  ストリーミング部分
#### app.py の gen_video 関数
今回は，簡単に言うと取得したカメラの１フレームをJPEG画像としてブラウザ上に表示します。そして，そのブラウザ上の画像部分だけを任意のタイミングで更新して動画にします．

この関数は，1フレームごとのJPEG画像を連続で持つイテレータを生成します．

#### app.py の video_feed 関数
紐づいたURLが呼ばれた際に，`gen_video` 関数で生成したイテレータをResponse オブジェクトとして返します．

### stream.html
#### {{url_for("video_feed")}}
これでページ内で"video_feed"のURLに紐づいた関数を呼び出します．
これをhtmlで`<img>`に埋め込むことで`Response`オブジェクトで返ってくる画像を表示します．

### kill_camera
通常カメラは1つしか起動できませんので，ブラウザで複数開こうとするとエラーになります．また，カメラを修了せずに切断してもエラーが出ます．これを回避するために，`kill_camera`関数ではカメラを終了し，その結果を報告するページを表示するようにしてあります．



### 備考
Responceオブジェクト等，Flaskに関する詳細については[Flaskドキュメント](https://flask.palletsprojects.com/en/2.0.x/api/)をご覧ください.


## トラブルシューティング
以下のような出力があり，`/stream`でビデオが表示されなくなった時は，
![](https://i.imgur.com/It3xvMd.png)

一旦コンテナとの接続を切り，
コンテナのホストマシン(jetson)のターミナルにて,以下のコマンドを実行します．
```
sudo systemctl restart nvargus-daemon
```
続いてコンテナを再起動して再度`app.py`を実行してみてください．
