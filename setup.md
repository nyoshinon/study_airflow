# セットアップ手順

※本手順書で記載している内容は、2022年2月時点での内容です。
利用サービス、アプリのアップデートにより、見た目や動作などが変更になる可能性がありますので、ご注意ください。
最新の情報は、参考に記載しているそれぞれのURLをご参照ください。

---

## EC2セットアップ

本手順書では、EC2 - Ubuntu20.04 のセットアップ方法について記載します。

### EC2作成

インスタンスを起動をクリック

1. AMI の選択
    1. Ubuntu20.04を選択
2. インスタンスタイプの選択
    1. t.xlargeを選択
3. インスタンスの設定
    1. 特に設定変更なし
4. ストレージの追加
    1. 30GBに設定
5. タグの追加
    1. 特に設定変更なし
6. セキュリティグループの設定
    1. 新しいセキュリティグループを設定する場合は下記を設定
        1. カスタムTCPルール, ポート範囲 8080 を設定
        1. SSH, ポート範囲 22 を設定
7. 確認

※インスタンスタイプ、ストレージ、ポートなどの設定は一例ですので、適時変更しても問題ありません。

### EC2ログイン

- 要: Teratermなどのターミナルソフト

※本手順書では、Elastic IPによるIPアドレスの固定を行っていないため、EC2の停止・起動ごとにパブリック IPv4 アドレスが変更になります。

1. ホストに作成したEC2のパブリック IPv4 アドレスを入力してOKをクリック
2. 初めてのアクセス時、セキュリティ警告がでますが、続行をクリック
3. ID, PW と秘密鍵を指定してOKをクリック

IDとPWのデフォルト値は以下の通り

- ID: ubuntu
- PW: 空欄

ubuntu以外を利用する場合のIDとPWは、AWSのマニュアルをご参照ください

- 参考: Amazon Linux インスタンスでのユーザーアカウントの管理
    - https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/managing-users.html




---

## Dockerインストール

- 参考: https://docs.docker.com/engine/install/ubuntu/

本手順書では、UbuntuにDockerをインストールする手順を記載します。

### Dockerインストール前の準備

まず、Dockerインストールの前に必要なパッケージをインストールします。

```bash
sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

GPGファイルをダウンロードします。

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Dockerのリストを作成します。

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Dockerインストール開始

### 最新版のインストール

```
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
```

### 特定バージョンのインストール

下記を実行し、利用できるバージョンのリストを表示します。

```
apt-cache madison docker-ce
```

利用したいバージョンをコピーしたら、下記コマンドの`<VERSION_STRING>`に置き換えて実行し、インストールします。

```
sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
```

### Docker動作確認

```
sudo docker --version
sudo docker run hello-world
```
---

## Docker-compose インストール 

- 参考: https://docs.docker.com/compose/install/

下記コマンドで、docker-composeをダウンロードします。

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

ダウンロードしたdocker-composeに実行権限を付与します。

```bash
sudo chmod +x /usr/local/bin/docker-compose
```

下記を実行し、docker-composeが実行できることと、バージョンを確認してください。

```bash
docker-compose --version
```

もし実行できない場合は、`usr/bin/`にシンボリックリンクを追加してください。

```bash
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

---

## Airflowセットアップ

- 参考: https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html

### Airflow : docker-composeのダウンロードと変更

※リポジトリ内にある `docker-compose.yaml` を利用せず、自前で用意する場合は下記にてダウンロードと内容の変更をしてください。

下記コマンドでAirflowのdocker-composeファイルをダウンロードします。

```bash
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml'
```

### Airflow: 初期化

必要なディレクトリと環境変数を追加します。

```bash
mkdir -p ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

Airflowを初期化します。

```bash
sudo docker-compose up airflow-init
```

初期化時に下記エラーが表示された場合、docker-compose のバージョンが古い可能性があります。docker-compose のバージョンが古くないか確認してください。

```log
ERROR: The Compose file './docker-compose.yaml' is invalid because:
Unsupported config option for services.airflow-cli: 'profiles'
Invalid top-level property "x-airflow-common". Valid top-level sections
 for this Compose file are: version, services, networks, volumes,
 and extensions starting with "x-".
```

docker-compose インストールで検索すると、日本語のマニュアルがトップに表示されます。2022年2月時点でダウンロードしてくるdocker-composeが古いものが記載されているため、上記のようなことが発生します...。マニュアルを利用するときは、いつのバージョンで作成されたものなのか、インストール後に、インストールしたバージョンがいくつか確認するようにしましょう。

### Airflow: 起動、停止

下記でコンテナを立ち上げ、Airflowを起動します。

```
sudo docker-compose up -d
```

`http://{IPアドレス}:8080`にブラウザからアクセスできることを確認してください。

ログインページが表示されたら、ID, PWともに `airflow` でログインできます。

これでAirflowを利用する準備が整いました。お疲れさまでした。

Airflowを落とす場合は下記を実行してください。

```
sudo docker-compose down -v
```
