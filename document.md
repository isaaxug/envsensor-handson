# オムロン環境センサ・Ambient・isaaxハンズオン

BLEセンサーデバイスとラズパイを使ってセンサーデータの可視化をしてみましょう。BLEデバイスにはオムロン環境センサ(2JCIE-BL01)、センサーデータの可視化にはAmbient.ioを使っていきます。

- [オムロン環境センサ - 2JCIE-BL01](https://www.omron.co.jp/ecb/product-info/sensor/iot-sensor/environmental-sensor)
- [Ambient - IoTデータ可視化サービス](https://ambidata.io/)

_※ isaax勉強会では必要なライブラリのインストール時間を節約するためにあらかじめ用意したSDカードを使用します。公式のRaspbianを使って1からセットアップする場合はこの記事の最後にある付録をご覧ください。_

## サンプルコードのフォーク

isaaxを使ってデバイスに配信するプログラムはGitリポジトリ(以下、リポジトリ)として管理します。今回はあらかじめ用意したサンプルコードをフォークして使います（フォークしたくない場合は、クローンして新規リポジトリとして作成してください）。

> リポジトリとは、ファイルやディレクトリの変更履歴を保持したプログラムファイルの集合です。Gitを使って開発する場合、大抵は1つのアプリケーションごとに1リポジトリ作成します(もちろん例外もあります)。

- [サンプルコード - GitHub](https://github.com/isaaxug/envsensor-ambient)

![fork repository](images/fork-repository.png)

## isaaxプロジェクトの作成

前節で自分のGitHubアカウントにフォークしたリポジトリを使ってisaaxにプロジェクトを作成します。isaaxアカウントをまだ作成していない場合はこのタイミングで登録しましょう(GitHubアカウントを使って登録すると便利です)。

- [isaax.io - 公式ページ](https://isaax.io/)

「新規プロジェクト追加」から新しいプロジェクトを作成します。

![create project](images/project-creation.png)

下図のように設定し、「保存」ボタンをクリックします。

![project settings](images/project-settings.png)

## デバイスの登録

作成したプロジェクトにデバイスを登録します。登録するにあたって、ラズベリーパイ側にはisaaxdというソフトウェアをインストールする必要があります。isaaxdは私たちがデバイスにインストールするアプリケーションの管理（起動、停止、ログの収集等）を担っています。

プロジェクトの作成後、下図のような画面が表示されます。プロジェクトトークンはデバイスにisaaxdをインストールする際に必要となります。インストールスクリプトはそのトークンを引数としてisaaxdのインストールをワンコマンド実行するためのスクリプトです。背景が黒い方の文字列をコピーしてください。

![project-token](images/project-token.png)

コピーしたコマンドはラズベリーパイ上で実行します。WindowsならPuttyやTeraTerm、MacやLinuxならターミナルを使ってラズパイにSSH接続しましょう。ユーザー名は`pi`、パスワードは`raspberry`です。

ログインに成功したら、コピーした文字列をラズパイ上で実行します。

![isaaxd installation](images/isaaxd-installation.png)

上図のように`isaaxd installation complete`が表示されればインストール成功です。

## isaaxを使ったログデータの確認

isaaxdのインストールが完了した時点でサンプルコードも同時にインストールされています。サンプルコードは、オムロン環境センサのアドバタイズパケットからセンサーデータを取得し、その中から照度を標準出力するシンプルなスクリプトです。isaaxdはこのアプリケーションの標準出力と、標準エラー出力を監視してクラウドにログデータとして送信します。

[サンプルコード(envsensor-ambient/sample1.py) - GitHub](https://github.com/isaaxug/envsensor-ambient/blob/master/sample1.py)

ダッシュボードに戻り、登録したデバイスの状態を確認しましょう。プロジェクトトークンのモーダルを閉じ、クラスターをクリックします。

> クラスターはプロジェクトに対して複数作成することができ、その下に紐付くデバイスをグループごとに分けるための機能です。この機能によって、開発中のデバイスと本番のデバイスを切り分けるといった応用的な操作が可能になります。

![cluster](images/cluster.png)

クラスターページを開くと「最近のデバイス」に先ほど登録したデバイスが表示されます。「バージョン」はインストールされているisaaxdのバージョンを、「リビジョン」はインストールされているアプリケーションのコミットID(Gitで作成した履歴ごとに付与されるID)をそれぞれ示しています。デバイスをクリックしましょう。

![device](images/device.png)

下図がデバイスの詳細ページになります。赤色の丸で囲われたボタンをクリックするとインストールしたアプリケーションのログを確認することができます。ラズベリーパイが期待する動作をしないときなどはこちらにエラーログが上がっている可能性が高いので確認しましょう。そのほか、割り当てられているIPアドレスの確認やアプリケーションの起動・停止などの機能を備えています。

![device-log](images/device-log.png)

この時点では、デバイスログは`No sensors found.`と表示されています。サンプルコードでは環境変数から読み取ったMACアドレスを元にBLEデバイスの判別とセンサー値の取得をおこないます。次節でisaaxを使ってユーザーアプリケーションがアクセスできる環境変数を定義する方法について説明します。

## 環境変数サービス (ユーザー変数)

isaaxではAPIキーのような認証情報や環境によって異なるエンドポイントなど、ハードコーディングしたくないデータを切り分けてデプロイする機能があります。isaaxで登録したこのデータはデバイス上で環境変数としてアクセスできます。

クラスターページから「Cluster Settings」をクリックしてドロップダウンを開きます。

![cluster settings](images/cluster-settings.png)

「ユーザー変数」タブから「＋環境変数追加」をクリックします。

![list of user var](images/list-of-user-var.png)

サンプルアプリケーションでは環境変数`BLUETOOTH_DEVICE_ADDRESS`からMACアドレスを取得します。値はそれぞれ手元の環境センサのMACアドレスに置き換えてください（勉強会では配布した資料にMACアドレスが記入してあるので、環境センサに付けられた番号から参照してください）。

![user var creation](images/user-var-creation.png)

保存後、「restart」ボタンをクリックし、デバイスログが`No sensors found.`から`Illumination: 254 lx`のような数値に置き換わっていれば成功です。ここでは照度を表示しています。スマホのライトなどでセンサーを照らしてみましょう。

![restart app](images/restart-app.png)

[デバイス上で動作するサンプル側](https://github.com/isaaxug/envsensor-ambient/blob/master/sample1.py#L12)では、11行目でMACアドレスを取得しています。

---

isaaxdのインストール以降、プロジェクトに紐づけたリポジトリに更新があるたびにデバイスを自動的にアップデートしてくれます。ここで、いくつかの課題にチャレンジしてisaaxを使ってデバイスを遠隔から更新する方法について学びましょう。フォークしたリポジトリの編集はPC上にクローンして行っても、GitHub上から直接おこなっても構いません。

## 課題1 どんなセンサーデータがあるか確認しよう

`sample1.py`の[22行目のコード](https://github.com/isaaxug/envsensor-ambient/blob/master/sample1.py#L22)が実際に環境センサから取得しているデータです。Pythonの組み込み関数`vars()`を使って、センサーデータにどのような種類があるのか確認しましょう。

[Python組み込み関数 vars() - 公式ドキュメント](https://docs.python.jp/3/library/functions.html#vars)

成功すると下図のような出力が得られます。

![answer task1](images/answer-task1.png)

## 課題2 任意のセンサーデータを出力しよう

課題1で確認したセンサーデータの中から好きなものを選んで、表示させるように変更を加えましょう。

---

## Ambientアカウント登録

次に、センサーデータをクラウドで可視化しましょう。今回はIoTのデータ可視化サービスであるAmbientを使っていきます。ArduinoスケッチやPythonモジュールからとても手軽に可視化を実現することができます。

- [サンプル - 環境センサを使った可視化の例](https://ambidata.io/ch/channel.html?id=5279)

それではアカウント登録をおこないます。下記リンクをクリックして、右上の「ユーザー登録(無料)」から登録作業をおこないます。

- [Ambient - IoTデータ可視化サービス](https://ambidata.io/)

メールアドレスとパスワードを設定すると入力したアドレス宛てにE-mailが届くので、記載のリンクをクリックして登録を完了しましょう。

## チャネル作成

Ambientアカウントに登録後、「Myチャネル」->「チャネルを作る」ボタンからチャネルを新規作成します。

![channel creation](images/channel-creation.png)

作成したチャネル情報の中から、データを保存するために「チャネルID」と「ライトキー」を後ほど使います。

![channel details](images/channel-details.png)

## 環境変数の追加（チャンネルID, ライトキー）

更新するサンプルアプリケーションがAmbientにデータを送信するために、チャネルIDとライトキーが必要となります。これらの情報を先ほどの容量で環境変数`AMBIENT_CHANNEL_ID`、`AMBIENT_WRITE_KEY`として設定します。

![extra vars](images/extra-vars.png)


## isaax.jsonのエントリーポイント変更

最後に、isaaxを使ってラズベリーパイ側のアプリケーションをAmbientにデータを送信するように変更します。サンプルコードはすでに用意してあるのでisaax.jsonを修正して実行するファイルを`sample1.py`から`sample2.py`に変更します。

フォークしたリポジトリの`isaax.json`9行目を下記のように変更してコミットします。

```json
{
  "name": "Envsensor Ambient",
  "version": "",
  "description": "",
  "author": "",
  "license": "",
  "language":"python",
  "scripts": {
	"start": "python3 -u sample2.py"
  }
}
```

Ambientのダッシュボードから該当するチャネルをクリックすると、数十秒後にチャートが自動的に作成されます。

![ambient dashboard](images/ambient-dashboard.png)

---

## 課題3 チャネルに複数のセンサーデータを送ろう

AmbientのPythonモジュールのドキュメントを参考に`sample2.py`に変更を加え、チャネルに複数のセンサーデータを送信するようにアプリケーションを更新しましょう。

[Ambient Pythonモジュール - 公式ドキュメント](https://ambidata.io/refs/python/)

---

## 付録1. SDカードのセットアップ

勉強会で使用したOSは2018-06-27リリースのRaspbian Stretch Liteをベースに作成しました。

- [Raspbian - ダウンロードページ](https://www.raspberrypi.org/downloads/raspbian/)

サンプルコードを動作させるには下記のライブラリをインストールしてください。

```
sudo apt-get install -y libperl-dev
sudo apt-get install -y libgtk2.0-dev
sudo apt-get install -y libglib2.0
sudo apt-get install -y libbluetooth-dev libreadline-dev
sudo apt-get install -y libboost-python-dev libboost-thread-dev libboost-python-dev
sudo apt-get install -y python3-pip
sudo pip3 install pybluez
sudo pip3 install pygattlib
sudo pip3 install git+https://github.com/AmbientDataInc/ambient-python-lib.git
```

## 付録2. 事後学習のために

isaaxについてより深く知るためには、公式のドキュメントを読んだり、コミュニティで質問してみましょう。

- [isaaxではじめるIoTの第一歩！ – Isaaxキャンプ](https://camp.isaax.io/hc/ja/articles/360001411588)
- [Isaaxとは - isaax公式ドキュメント](https://isaax.io/docs/)
- [コミュニティ – Isaaxキャンプ](https://camp.isaax.io/hc/ja/community/topics)

無料の勉強会も開催しているので、ハンズオンしたい方はこちらにもご参加ください。

- [isaax User Group - connpass](https://isaaxug.connpass.com/)

SNSでもIoTに関する情報を発信しています。フォローお願いします。

- [XSHELL (@xshell_inc) | Twitter](https://twitter.com/xshell_inc)
- [Xshell - Facebookページ](https://www.facebook.com/xshellinc)