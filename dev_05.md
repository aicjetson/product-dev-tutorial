# Flaskを使ったwebアプリケーションの作成2　　ストリーミングで物体検出

## 推論に適したJetson Nano
深層学習には「学習」と推論の2つのパートが存在します．「学習」は大量のデータをニューラルネットワークに入力し，その重みやバイアスといったパラメータを最適化していくことで精度を向上させていく段階です．ニューラルネットワークの各パラメータの数は膨大であり，これを計算するには高い計算能力を持ったコンピュータが必要になってきます．
Jetson Nanoは472GFLOPSの計算能力をもちますが，これでも学習を行うには不十分です．
以上の理由から，学習には高性能なGPUを搭載したサーバ型のコンピュータが必要となってきます．

一方で，推論においては，学習のような高い計算能力は求められません．しかしながら，推論システムは常時動作させたり，リアルタイムに出力を行ったりする必要があるため，消費電力の低さ，設置環境の柔軟性などが求められます．
Jetsonは  こうした「推論」に適しています．


## YOLOとは
### YOLOの特徴
YOLO(You Only look Once)は物体検出アルゴリズムの1つです．

物体検出とは，画像内において物体が存在する領域とそのクラス分類を行うタスクになります．

YOLOでは，物体が位置する領域の推論と，その物体が何であるかの推論を1つのCNNで処理します．

かつては，画像内でどこに物体が存在するかを推論し，それを四角いBoxで囲み，そのBoxの中に存在する物体をクラス分類することで物体検出を行う，という二段構成が一般的でした．しかしこれでは物体領域として推論された領域全てにクラス分類を行う必要があるため，処理に非常に時間がかかります．
これらの例としてはR-CNNといった手法が挙げられます．

![](https://i.imgur.com/tZgBqep.png)
R-CNN (R. Girshick, CVPR, 2014)

これに対してYOLOは物体の領域提案とクラス認識を1つのニューラルネットワークで処理することができます．

![](https://i.imgur.com/D2PQLv1.png)
YOLO (J. Redmon, CVPR, 2016)

今回は，そのバージョン2で，軽量版である「Tiny YOLO v2」を使います．

### YOLOアルゴリズム概要
まず，入力画像をS×S領域に分割します．

![](https://i.imgur.com/h8okkON.png)

#### 物体の位置の推定
ここで分割された各gird cellに対してB個のBounding Box を推定します.

1つのBounding Boxに座標値(x,y,w,h)とそのBox内に物体が存在する（背景ではない）信頼度スコアの5つの値が出力されます．

座標値のx, yはgrid cellの境界を基準にしたBounding Boxの中心座標、幅wと高さhは画像全体のサイズに対する相対値です．
 
 信頼度スコアは物体が存在する確率を示します．


#### 物体の種類の推定

各grid cell単位で物体の種類も推定します．

C種類のクラスに関して，あるgrid cell が背景ではなく物体である場合にどのクラスに属するか，という条件付き確率を推定します．

![](https://i.imgur.com/UYtwQWh.jpg)

#### 物体検出としての出力を導く
以上の2パートで推定した「Bounding Box」と各Grid Cellのクラスを合わせることで，物体のクラス情報を持つBounding Boxを得ることができます．

#### 非最大値抑制
上までの処理だと，ある1つの物体を検出するBounding Box が複数存在してしまいます．

以下の概念を導入してこの問題を解決します．

- IOUスコア
2つのBoxがどのくらい重なっているかを表す指標として，IOU(INtersection over Union)と呼ばれる値があります．
この値が大きいとより一致しているということになります．
![](https://i.imgur.com/j9wIdl6.png)

IOUスコアがある閾値より大きいものは全て同じ物体を検出しているとみなし，その中で最も推論の際のスコアが高かったものを選び，それ以外のBoxは破棄します．



以上がYOLOによる物体検出の流れになります．

今回は既存のライブラリを用いるので，こちらは実装しませんが，自分で使うアルゴリズムはぜひ理解しておきましょう．

 
## Tiny YOLO v2
改良されたYOLOであり，その軽量版です．
先述したYOLOではグリッドセルにより物体とそのクラスを検出しましたが，YOLO v2はグリッドの代わりに「アンカーボックス」と呼ばれる概念を使用しています．今回はYOLO v2そのものを実装するのではなく，これをJetson Nano上で動かすことを目的としているため，この元になっているYOLOの理解までにとどめます．

## Jetson Nanoで深層学習の推論を実行するために

### ONNX

先ほど説明したように，Jetson Nanoが得意なことは推論ですから，今回物体検出で扱うモデルはすでに学習済みのものを扱います．

深層学習の学習済みのモデルのデータ形式はフレームワークごとに異なることが多いですが，最近は共通の形式が使われ始めましたその代表的なものが，

「Open Neural Network Exchange(ONNX)」と呼ばれるFacebook とMicrosoftが中心となって開発したコミュニティプロジェクトです．

このONNX形式に学習したモデルを一旦変換してしまえば，深層学習フレームワーク間でニューラルネットワークモデルに互換性を持たせることができます．

### TensorRT
深層学習推論のためのプラットフォームでNVIDIA    社が提供しているものに「TensorRT」と呼ばれるものがあります．

これはNVIDIAのGPU向けに最適化して実行が可能です．

- 深層学習フレームワークが生成した学習済みのモデルをインポート
- TensroflowからTensorRTを使用
- TensorRT APIを利用

以上の3パターンの利用の仕方があります．

#### TensorRTのワークフロー

TensorRTで利用できる学習済みモデルの形式は「ONNX」,「Caffe」，「Tensorflow」の3形式です．

それぞれの形式のモデルを読み込み，最適化してTensorRT形式のネットワークモデルに変換します．
このTensorRT形式は「TensorRTプラン」などと呼ばれます．

この処理に時間が少々かかりますが，一旦TensorRT形式にしてしまえば，NVIDIA GPU向けに最適化していますから，比較的高速な推論が可能になります．

## 物体検出webアプリケーションの構成

構成として前回と変わるのは，ストリーミングする各フレームごとの画像をレスポンスとして返す前に，物体検出を行い，以下のように検出結果を画像に書き込んだ上で出力するということだけです．

![](https://i.imgur.com/6NKsi19.png)

前回の構成↓
![](https://i.imgur.com/lLmiseN.png)


今回の構成↓
![](https://i.imgur.com/9AQRmw4.png)

## 前準備

### junnbi
前回作成したFlask用のコンテナはCSIカメラに合わせたdocker runを行いました．今回はUSBカメラを使うので別のコンテナを作成しましょう．
まず，前回のディレクトリを複製します．
前回のディレクトリはhomeディレクトリにあるはずなので，homeディレクトリに移動したら，
```
$ cp -r streaming_app yolo_app
```
として複製しましょう．
そして中に入ったら`docker_run.sh`を以下のように書き換えてください．
```=
#! /bin/bash

docker run -it \
--runtime nvidia \
--network host \
-v ~/yolo_app:/usr/app \
--device /dev/video0 \
--name flask_yolo_app_con flask_uwsgi_nginx:stream 
```
コンテナを立ち上げる前に以下のことを行っておきましょう．
#### クロックアップ
```
$ sudo nvpmodel -m 0
$ sudo jetson_clocks
```
#### nvargusデーモンの再起動
```
$ sudo systemctl restart nvargus-daemon
```

以上が終了したらコンテナを起動してコンテナにアクセスしましょう．

```
$ bash docker_run.sh
```

以降は特筆しない限り，コンテナの中での操作になります．

### nginxとuWSGIの設定
先程のDockerファイルにはnginxとuWSGIのインストールが記述されていました．
これらはflaskのいわゆる本番環境を構築するためにインストールしました．
前回までは，`python3 app.py`と実行して動き出す開発用の即席サーバ内でflaskを動かしていましたが，この開発用サーバはさまざまな制限があります．
今回扱う`pycuda`といった深層学習推論をGPUで行うために必要になるライブラリは開発用サーバでは正常に動きません．

従って，今回は本番環境をuWSGIとnginxを使って構築しますが，講習会の本質から外れるため，おまじないだと思って以下の作業を行ってください．


コンテナで以下を実行します．

```
# cd /etc/nginx/sites-available
# cp default ./app_nginx.conf
```
これで作成された`app_nginx.conf`を開き，中身を一旦全て削除してから以下の内容をコピペしてください．

```
# Virtual Host configuration for example.com
server {
        listen 80;
        server_name example.com;        
        location / {
          include uwsgi_params;
          uwsgi_pass unix:/tmp/uwsgi.sock;
        }
}
```
こちらを実行してください．

```
# cd /etc/nginx/sites-enabled/
# ln -s /etc/nginx/sites-available/app_nginx.conf app_nginx.conf
```

`/etc/nginx/nginx.conf`について，以下の内容が書かれた場所を探し，

```
# Virtual Host Configs
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;  
```
以下に変更してください．

```
# Virtual Host Configs
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*.conf; 
```
同じファイル内で，以下の画像のように`Basic Settings`とコメントが書かれた部分の下を画像と同じになるように変更して，timeoutまでの時間を延長しましょう．
![](https://i.imgur.com/yESFSrZ.png)




`app.py`と同じ階層(yolo_appディレクトリ直下)に以下のファイル"`app.ini`"を作成してください，
```
[uwsgi]
module = app
callable = app
master = true
harakiri = 600
processes = 1
socket = /tmp/uwsgi.sock
chmod-socket = 666
vacuum = true
die-on-term = true
chdir = ~/yolo_app  # フォルダのディレクトリを指定する
```

### 追加のパッケージインストール
`requirements.txt`を`usr/app`直下に作成し，以下を書き込んでください．

```
numpy>=1.15.1
onnx>=1.6.0
pycuda>=2017.1.1
Pillow>=5.2.0
wget>=3.2
paho-mqtt
```
以下を実行して上のパッケージをインストールしましょう．
```
# pip3 install -r requirements.txt
```

## 前回作成したwebアプリにYOLO機能を追加
これ以降はソースコードをGithubにあげているのでそちらを参照しながら読み進めてください．
（https://github.com/aicjetson/yolo_streaming）
それでは機能を追加していきます．

### モジュール
まずは物体検出機能のために必要な処理の部品が書かれているモジュールを見ていきます．

#### yolo_tool.py
まずはYOLOに必要なモジュールを作成します．
`yolo_tool.py`にはYOLOのアルゴリズムに直接関わるわけではないけれども，yoloを実行する上で必要になる処理を行う関数が記述されています．これらはメインとなる処理が書かれるファイル内で呼び出されます．

- `draw_bboxes()` : 物体検出処理を行った画像にバウンディングボックスを描画する
- `draw_message()` : 文字列を画像に埋め込んで情報を表示する
- `reshape_image()` : OpenCVで取り込んだ画像を，YOLOのネットワークに入力するために変換を行う
- `download_file_from_url()` : URLを指定してファイルをダウンロードする．ダウンロードしてある時は行わない
- `download_label()` : クラスラベルのファイルをダウンロードする
- `download_model()` : ONNXの学習済みモデルをダウンロードする． ダウンロード後はアーカイブファイルを展開する．

#### data_processing.py
YOLOに入力するための画像の前処理クラス．そして出力結果に対する後処理を行うクラスが格納されています．

こちらはNVIDIAが公開しているTensorRTでYOLOを扱う際のテンプレートになります．

#### get_engine.py
TensorRTにモデルを読み込む処理が記述されています．
こちらもNVIDIAが公開しているTensorRTを使うためのテンプレートです．
#### common.py
こちらも同様にNVIDIAが公開しているTensorRTを使うためのテンプレートです．`pycuda`と呼ばれるライブラリを使ってGPUを使った推論を行うための処理が書かれたモジュールとなっています．

### メインの処理部分
まずは`app.py`を見てみます．`gen_video`を以下のように書き換えます．（引き続きGithubにファイルがあるのでそちらでソースコードを見ていただいても構いません．）
先ほど確認した`get_engine.py`からimportsして`TensorRT`のモデル読み込みを行っていることがわかるかと思います．

```python=
def gen_video(camera):
    # TensorRTにモデルを読み込む
    with get_engine(
        camera.onnx_file_path, camera.engine_file_path
    ) as engine, engine.create_execution_context() as context:

        # TensorRT用にバッファメモリを割り当て
        ctx = cuda.Context.attach()
        inputs, outputs, bindings, stream = common.allocate_buffers(engine)
        ctx.detach()

        fps = 0.0
        frame_count = 0

        while True:
            # フレームの開始時刻を記録
            start_time = time.time()

            # 物体検出が完了してBBOXや各種情報が書き込まれた画像を取得する．
            frame = camera.get_detection(
                inputs, outputs, bindings, stream, fps, frame_count, context
            )

            # フレームのイテレータの作成
            if frame is not None:
                yield (
                    b"--frame\r\n"
                    b"Content-Type: image/jpeg\r\n\r\n" + frame.tobytes() + b"\r\n"
                )
            else:
                print("frame is none")

            # 1フレームの処理時間を計測してFPS値を算出する
            elapsed_time = time.time() - start_time
            time_list = np.append(camera.time_list, elapsed_time)
            time_list = np.delete(time_list, 0)
            avg_time = np.average(time_list)
            fps = 1.0 / avg_time

            frame_count += 1

```

`camera.py`は以下のように`Camera`クラスのコンストラクタ`__init__`にYOLOで必要になる各種パラメータを追加しています．さらに，`get_detection`メソッドを追加し，YOLOによる物体検出を行い，画像に物体検出結果を書き込んだ結果を返り値としています．

(以下は抜粋です．)


```python=
.
.
.

    def __init__(self):
        self.width = 1920
        self.height = 1080
        self.video = cv2.VideoCapture(0, cv2.CAP_V4L)

        self.video.set(cv2.CAP_PROP_FRAME_WIDTH, self.width)
        self.video.set(cv2.CAP_PROP_FRAME_HEIGHT, self.height)
        self.act_width = self.width
        self.act_height = self.height
        self.frame_info = "Frame:%dx%d" % (self.act_width, self.act_height)

        # Download the label data
        self.categories = download_label()

        # Configure the post-processing
        self.postprocessor_args = {
            # YOLO masks (Tiny YOLO v2 has only single scale.)
            "yolo_masks": [(0, 1, 2, 3, 4)],
            # YOLO anchors
            "yolo_anchors": [
                (1.08, 1.19),
                (3.42, 4.41),
                (6.63, 11.38),
                (9.42, 5.11),
                (16.62, 10.52),
            ],
            # Threshold of object confidence score (between 0 and 1)
            "obj_threshold": 0.6,
            # Threshold of NMS algorithm (between 0 and 1)
            "nms_threshold": 0.3,
            # Input image resolution
            "yolo_input_resolution": INPUT_RES,
            # Number of object classes
            "num_categories": len(self.categories),
        }
        self.postprocessor = PostprocessYOLO(**self.postprocessor_args)

        # Image shape expected by the post-processing
        self.output_shapes = [(1, 125, 13, 13)]
        self.onnx_file_path = download_model()

        self.engine_file_path = "model.trt"

        self.time_list = np.zeros(10)
        
.
.
.

    def get_detection(
        self, inputs, outputs, bindings, stream, fps, frame_count, context
    ):

        # カメラからフレームキャプチャ
        ret, img = self.video.read()

        # Tiny YOLO v2入力にあわせてフレームを変換
        rs_img = cv2.resize(img, INPUT_RES)
        rs_img = cv2.cvtColor(rs_img, cv2.COLOR_BGRA2RGB)
        src_img = reshape_image(rs_img)

        # TensorRTを使って推論実行する
        inputs[0].host = src_img
        trt_outputs = common.do_inference(
            context, bindings=bindings, inputs=inputs, outputs=outputs, stream=stream
        )

        # 後処理用に出力データをネットワークの出力データ形状を変換する．
        trt_outputs = [
            output.reshape(shape)
            for output, shape in zip(trt_outputs, self.output_shapes)
        ]

        # 後処理を実行してBBOXを計算する．
        boxes, classes, scores = self.postprocessor.process(
            trt_outputs, (self.act_width, self.act_height)
        )

        # BBOXを画像に描画する．
        if boxes is not None:
            draw_bboxes(img, boxes, scores, classes, self.categories)
        if frame_count > 10:
            fps_info = "{0}{1:.2f}".format("FPS:", fps)
            msg = "%s %s" % (self.frame_info, fps_info)
            draw_message(img, msg)
        if frame_count == 10:
            cv2.imwrite("./img.jpg", img)

        ret, frame = cv2.imencode(".jpg", img)

        return frame
```

## 実行


今回は開発用サーバではなく， uWSGIとNginxを使って動かします．いつもの以下のコマンドで実行すると開発用サーバが立ち上がってしまいます．
```
# python3 app.py
```

### uWSGIとNginxを使った実行
まずターミナルを2つ開いてください．まず１つ目で以下を実行し，uWSGIを起動します．

```
# uwsgi --ini myapp.ini
```
pythonのエラー文が出力されることなく以下のような出力が表示されれば成功です．

```
spawned uWSGI master process (pid: 7263)
spawned uWSGI worker 1 (pid: 7322, cores: 1)
```
つづてもう1つ開いたターミナルで以下を実行することでNginxを起動します．
```
# /etc/init.d/nginx start
```

ここでVScodeで実行を行っていると，port forwardingが行われ，Jetsonの80番ポートが作業PCのポート（番号は人によって異なるかと思います）に自動でフォワーディングされます．しばらく待っても自動でフォワーディングされない時は自分でフォワーディングしてしまっても構いません．

アクセスするとこのページが開きます．

![](https://i.imgur.com/cwtLwEq.png)

ストリーミング開始のリンクを踏むとしばらくローディングに入ります．
これはモデルのダウンロードを行っているからです．しばらくかかります．
uWASGIを起動したターミナルに進行状況が出力されるのでそこで確認してください．
![](https://i.imgur.com/ly4Lfnb.png)

以下はロード中ページです，動画部分は表示されません，しばらく待ちましょう．
![](https://i.imgur.com/cKOulX0.png)
以下の出力が出てくると物体検出された動画が表示されます．
![](https://i.imgur.com/M84xvoA.png)
自分が映るようにカメラを調節してみてください．personと認識されるはずです．


### 参考
Jetson Nanoによる物体検出に関しては以下を参考に作成しています．([Amazonリンク](https://amz.run/4gm7))


```
からあげ, Jetson Nano超入門 ソーテック社, 2019
```

