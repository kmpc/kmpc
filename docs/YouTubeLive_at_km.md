# 縛り環境でYouTubeLive

## パターン一覧
- 外部に立てたHTTPサーバを経由する

![](https://asset1.chasoba.net/a/ytl_slide1.png)

- SSHで掘ったトンネルを経由する

![](https://asset1.chasoba.net/a/ytl_slide2.png)

- GSuite for Education のアカウントを利用して，Hangout Meet(双方向ビデオ通話) を利用する

    双方向ではあるものの，片方ミュートにしてしまえば一方通行にできる。メリットとしては，外部にサーバを立てなくてよい。すんなりとProxyを超えられる。

- Zoom を利用する(要申請)

    良くも悪くも話題の Zoom だが，セットアップが容易である。ただ，Proxyによってブロックされているかは不明。

## 外部に立てたHTTPサーバを経由するパターン
### 概要
配信PCから，細切れにした動画ファイルを外部(レンサバなど)に立てたHTTPサーバへ送信する。外部のサーバは，さきほどのHTTPサーバから動画ファイルをGETしYouTubeLiveのサーバへPOST(映像送出)する。

### 外部に立てる必要のあるサーバ
- piping-serverを立てられるサーバ
    - **役割：細切れにされた動画ファイルを受け取る(中継する)HTTPサーバ**
    - piping-serverのGitHub：[nwtgck/piping-server： Streaming Data Transfer Server over HTTP/HTTPS： designed for people using pipe in Unix-like OS and even for browser users](https://github.com/nwtgck/piping-server)
    - Deploymentにポータブル版があるのでroot権限無しでも立てられるかもしれない：
        - [Ecosystem around Piping Server · nwtgck/piping-server Wiki](https://github.com/nwtgck/piping-server/wiki/Ecosystem-around-Piping-Server#deployment)
         - [nwtgck/piping-server-pkg： Piping Server as an portable executable](https://github.com/nwtgck/piping-server-pkg#download--run)
    - Herokuにも立てられる(検証済み)。しかし，調子がいいときは安定して送出できるが，調子が悪いときは送出が途切れる場合がある。すなわち不安定。
    - **`https`ではなく`http`でアクセス可能なこと**(後述するffmpegの仕様上)
    - **`80` or `443`ポートでアクセス可能なこと**
- ffmpegを動かせるサーバ(piping-serverを立てられるサーバと同じサーバでよい)
    - **役割：piping-serverから細切れの動画ファイルをGETしてYouTubeLiveへPOSTする**
    - [Download FFmpeg](https://www.ffmpeg.org/download.html)
    - 確かポータブル版あり
### 手順
前提：外部にpiping-serverが立てられていること

1. Webカメラなどがつながっている送出用PCにてNginxを起動する(ファイルはGDrive `YTL/nginx_1.7.12.1_Lizard`)。

    `nginx.exe`が存在するディレクトリでcmd立ち上げて`nginx`でNginxが起動する。
2. 送出用PCにて，OBS(配信ソフト)を起動する(ファイルはGDrive `YTL/OBS-Studio-24.0.3-Full-x64/bin/64bit/obs64.exe`)。
    `obs64.exe`をタブルクリックで起動する。初回起動時はチュートリアルのようなものが表示されるが無視する。
3. 送出用PCにて，OBSにWebカメラの映像を取り込む。
    1. ウィンドウ下部の「ソース」の枠内を右クリ→「追加」→「映像キャプチャデバイス」をクリックする

        ここで，「ウィンドウキャプチャ」などを選択することでデスクトップ画面などの送出もできる
    3. 新規作成やら既存を追加やら書かれたウィンドウが表示されるが，そのまま「OK」をクリックする
    4. 「デバイス」でWebカメラらしきデバイスを選択して，「OK」をクリックする
    5. OSBのウィンドウ内にWebカメラの映像が映る
4. 送出用PCにて，OBSの送出用設定を変更する
    1. OBSの右下「設定」→「配信」→「サービス」→「カスタム」を選択する
    2. 「サーバ」に`rtmp://127.0.0.1/live`と入力する (localhostでもよい？)
    3. 「ストリームキー」に`test`と入力する (適当な文字列でもよい？)
    4. ウィンドウ左「出力」タブへ移動して，「出力モード」を「詳細」に変更する
    5. 「配信」タブ内中央あたりの「ビットレート」を`1000kbps`に変更する (アップロードの速度がどのぐらいでるかわからないので，とりあえず控えめに設定 推奨値は以下URL参照)
    6. 「キーフレーム間隔」を`2`に変更する

         - [ライブ エンコーダの設定、ビットレート、解像度を選択する - YouTube ヘルプ](https://support.google.com/youtube/answer/2853702?hl=ja)
    7. ウィンドウ左「映像」タブへ移動して「基本解像度」，「出力解像度」，「FPS」を必要に応じて変更する

        解像度は両方とも同じにしないと画面の縮尺がおかしなことになります
    8. ウィンドウ右下の「OK」or「適用」をクリックして保存し，設定ウィンドウを閉じる
5. 送出用PC，中継サーバでコマンドを打つ準備をする

    cmdにコマンドの文字列を入力するだけで，**まだ実行しない。**

    1. 送出用PCにて，細切れの動画ファイルを外部へPOSTするためのffmpegを準備する(ファイルはGDrive `YTL/ffmpeg/ffmpeg.exe`)
    `http://piping.example.com`は，piping-serverが立てられているサーバのアドレス
        ```
        ffmpeg.exe -re -i rtmp://127.0.0.1/live/test -vcodec copy -acodec copy -http_proxy http://proxy.hoge.jp:80 -f flv http://piping.example.com/yt.flv
        ```
    2. piping-serverが立てられているサーバもしくは適当な外部のサーバにて，細切れの動画ファイルをGETしてYouTubeLiveへPOSTするためのffmpegを準備する

        YouTubeLiveの管理画面からイベントを作成し，
        `streamkey`には，YouTubeLiveの管理画面から取得した「ストリーム名」をコピペする
        ```
        ffmpeg -i http://127.0.0.1/yt.flv -vcodec copy -acodec copy -f flv rtmp://a.rtmp.youtube.com/live2/streamkey
        ```
6. 以下の順番で，コマンドなどを実行して配信を開始する
    1. 送出用PCにて，OBSの「配信開始」をクリックする
    2. 外部のサーバにて，5.2のコマンドを実行する
    3. 送出用PCにて，5.1のコマンドを実行する
7. YouTubeLiveの管理画面で，映像が送出されていることを確認する
8. 以上

## SSHで掘ったトンネルを経由するパターン
### 概要
送出用PCから外部のサーバへSSH接続する。YouTubeLiveへ，外部のサーバを経由して送出されるようにする。

### 外部に立てる必要のあるサーバ
- SSH接続ができ，ffmpegを動かせるサーバ
    - **役割：送出用PCとYouTubeLiveの間を中継する。疑似VPNのような手法**
    - **`80` or `443`ポートでSSHサーバがホストできること(`22`はNG)**
    - ffmpegによって，送出用PCとYouTubeLiveサーバ間を中継する
    - [Download FFmpeg](https://www.ffmpeg.org/download.html)

### 手順
1. 送出用PCにて，TeraTermで外部のサーバへSSH接続(ポート：`443` or `80`)する(プロキシ指定忘れずに)
2. 送出用PCにて，TeraTermの「設定」→「SSH転送」→「追加」→以下を入力して→「OK」

    ローカルのポート：`11935`
    
    リモート側ホスト：`127.0.0.1`
    
    ポート：`11935`
3. 送出用PCにて，OBS(配信ソフト)を起動する(ファイルはGDrive `YTL/OBS-Studio-24.0.3-Full-x64/bin/64bit/obs64.exe`)。
    `obs64.exe`をタブルクリックで起動する。初回起動時はチュートリアルのようなものが表示されるが無視する。
4. 送出用PCにて，OBSにWebカメラの映像を取り込む。
    1. ウィンドウ下部の「ソース」の枠内を右クリ→「追加」→「映像キャプチャデバイス」をクリックする

        ここで，「ウィンドウキャプチャ」などを選択することでデスクトップ画面などの送出もできる
    3. 新規作成やら既存を追加やら書かれたウィンドウが表示されるが，そのまま「OK」をクリックする
    4. 「デバイス」でWebカメラらしきデバイスを選択して，「OK」をクリックする
    5. OSBのウィンドウ内にWebカメラの映像が映る
5. 送出用PCにて，OBSの送出用設定を変更する
    1. OBSの右下「設定」→「配信」→「サービス」→「カスタム」を選択する
    2. 「サーバ」に`rtmp://127.0.0.1:11935/live`と入力する
    3. 「ストリームキー」に`test`と入力する (適当な文字列でもよい？)
    4. ウィンドウ左「出力」タブへ移動して，「出力モード」を「詳細」に変更する
    5. 「配信」タブ内中央あたりの「ビットレート」を`1000kbps`に変更する (アップロードの速度がどのぐらいでるかわからないので，とりあえず控えめに設定 推奨値は以下URL参照)
    6. 「キーフレーム間隔」を`2`に変更する

         - [ライブ エンコーダの設定、ビットレート、解像度を選択する - YouTube ヘルプ](https://support.google.com/youtube/answer/2853702?hl=ja)
    7. ウィンドウ左「映像」タブへ移動して「基本解像度」，「出力解像度」，「FPS」を必要に応じて変更する

        解像度は両方とも同じにしないと画面の縮尺がおかしなことになります
    8. ウィンドウ右下の「OK」or「適用」をクリックして保存し，設定ウィンドウを閉じる
6. YouTubeLiveの管理画面から「エンコーダ配信」でイベントを作成し，ストリームキーを取得する
7. TeraTerm(外部のサーバ)にて，以下のコマンドを打つ

    `streamkey`は，管理画面で取得したストリームキーをコピペ
    ```
    ffmpeg -listen 1 -i rtmp://127.0.0.1:11935/live/test -c copy -f flv rtmp://a.rtmp.youtube.com/live2/streamkey
    ```
8. 送出用PCにて，OBSの「配信開始」をクリックする
9. YouTubeLiveの管理画面にて，映像が送出されていることを確認する
