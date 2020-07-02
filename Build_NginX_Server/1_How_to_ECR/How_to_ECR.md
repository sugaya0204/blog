はじめまして、サーバーワークス新人エンジニアの菅谷です。  
今年4月にサーバーワークスに新卒で入社したのですが、せっかく半年たったので、自分への覚書ついでにやったことをブログにしてみようと思います。

今回のテーマは「AWSサービスのECRとECSを使ってnginxのページを表示させよう」です。  
本ページでは前編として、ECRへのイメージのプッシュまでを行いたいと思います。

この2つのサービスを使用すると自分で作成したDockerイメージからコンテナを素早く簡単に起動できます。

# 事前知識
## Docker
コンテナを作成するサービス。CLIから簡単実行できる。

- コンテナ  
プロセスごとの仮想化により作成される個々の環境を指す。
必要な時に即座かつ容易に環境構築でき、壊すのも簡単。
- DockerFile  
どのようなコンテナを起動するか定義するファイル。
ゆえに冪等性がある。
- Dockerイメージ  
コンテナを作成する際に元となるイメージファイル。
前述のDockerFileをビルドすることで作成される。
Docker社が提供しているものや一般ユーザが公開してるものもある。

## ECR
AWSが提供するDockerイメージリモート管理サービス。  
後編で取り扱うECSと組み合わせればマネコンからでもローカルからでも迅速に環境構築ができる。

# ゴール
全編通してのゴールは以下です。

[How_to_ECR_1]

前編では、EC2上で準備したDockerfileをECRにPushするところをゴールとします。

# 作業
## 作業環境の構築
CloudFormationテンプレ―トを使用して、以下の環境を構築します。

[How_to_ECR_2]

使用するテンプレートファイルです

[cf_template.yaml]

CloudFormationの手順は[こちら]()のページにまとめているので、参考にしてください。

今回はスタック名を *docker-handson-<YYYYMMDD>* にして作成してください。

## IAMRole権限の付与
### IAMRoleの作成
Worker インスタンスでECRを操作するためのIAM Roleを作成します。

マネコンから IAM コンソールへアクセスします。

左ペインから *ロール* を選択して *ロール* の作成を押します。

各パラメータを選択して *次のステップ* へ。
|Key|Value|
|---|---|
|信頼されたエンティティの種類を選択|AWSサービス|
|一般的なユースケース|EC2|

*AmazonEc2ContainerRegistryFullAccess* を選択して *次のステップ:タグ* へ。

タグは特に挿入せず、*次のステップ:確認* へ。

以下パラメータを埋めて *ロールの作成* を押します。
|Key|Value|
|---|---|
|ロール名|<YYYYMMDD>-docker-handson-worker-role|

### IAMRoleのアタッチ
作成したIAMRoleをWorkerインスタンスにアタッチします。

EC2コンソールにて、左ペインの *インスタンス* を選択します。

一覧からインスタンス名 *<YYYYMMDD>-docker-handson-worker* を選択します。  
※この時点でインスタンスがない場合はCloudFormationのスタックがエラーを吐いていないか確認してください。

*アクション* > *インスタンスの設定* > *IAMロールの割り当て/置換* を選択します。

IAMロール欄に *<YYYYMMDD>-docker-handson-worker-role* を入れて *適用* を押します。

これで必要な権限の付与は完了です。

## Dockerイメージの作成
### Dockerfileの作成
WorkerインスタンスへSSHログインします。

dockerパッケージを入れ、起動します。
```bash
$ sudo yum -y update 
$ sudo yum -y install docker
$ sudo systemctl start docker
```

あらかじめ使う単語を環境変数に入れます。

```bash
$ export REGION_NAME=ap-northeast-1
$ export REPOSITORY_NAME=<YYYYMMDD>-docker-handson-repo
$ export -p //確認用
```

Nginxで表示させたいオリジナルのhtmlファイルを作成します。
```bash
$ vi new_index.html
```
```html
<p>Deployed Nginx content using Docker container!!</p>
```

続いてDockerfileの中身を定義します。
それぞれのパラメータは以下を参考にしてください。
|Key|Value|
|---|---|
|ARG|宣言された変数を格納する。FROMより前に宣言されたARGは、ビルドステージ内に含まれないためFROM以降で利用するには再宣言する必要がある|
|FROM|ベース・イメージを定義する。宣言することでビルド時に公開リポジトリからイメージを取得する。|
|MAINTAINER|Dockerfileの作者情報を記述する|
|RUN|ビルド時に実行されるコマンドを定義する|
|ADD|指定のファイルをコンテナ内に保存する|
|EXPOSE|コンテナの実行時にネットワーク所のどのポートをリッスンするか指定する|
|CMD|コンテナ起動時に実行されるコマンドを定義する|
参考: [Dockerfile リファレンス](http://docs.docker.jp/engine/reference/builder.html)

```bash
$ vi Dockerfile
```
```
ARG VERSION=2
FROM amazonlinux:$VERSION
MAINTAINER sugaya
RUN echo "now building..."
RUN amazon-linux-extras install nginx1 -y　#Nginxのversionは都度確認して変更してください。
ADD ./new_index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["/usr/sbin/nginx", "-g", "daemon off;"] #フォアグラウンドでNginxを起動する
```

※捕捉1. コンテナ内で起動するプロセスがバックグラウンド実行の場合、起動後すぐ停止してしまうので、必ずフォアグラウンドで実行しましょう。

参考: https://heartbeats.jp/hbblog/2014/07/3-tips-for-nginx-on-docker.html

※捕捉2. amazonlinux2イメージにデフォルトでsystemdが導入されていないため、systemctl ではなく直接nginxを起動させています。

参考: https://qiita.com/mach3/items/33f2b234babe679c759f

ここまで色々やりましたが、以下の構成になっていればOKです
```bash
$ ls
Dockerfile  new_index.html
```
### Dockerイメージの作成
出来上がったDockerfileをビルドしてDockerイメージを作成しましょう。

ビルドにはdocker image build コマンドを使います。  
"-t" はビルドイメージの名前を、"."はDockerfileのあるディレクトリを指しています。
```bash
$ sudo docker image build -t nginx-test .
$ sudo docker images //確認用
```

### Dockerイメージの動作確認
上で作ったDockerイメージを起動してみましょう。

コンテナの起動は docker container run で行えます。  
"-d"はコンテナプロセスをバックグラウンドで起動します。  
"-p <LocalPort>:<ContainerPort>" は、特定のローカルポートからきた通信を特定のコンテナポートへ誘導するようマッピング定義をします。
```bash
$ sudo docker container run nginx-test:latest -d -p 8080:80
$ sudo docker ps -a //プロセス起動確認
```
実際にWebブラウザでWorkerの8080ポートにアクセスしてみましょう。
*http://<PublicIPaddress>:8080* にアクセスして画面が映る確認しましょう。

動作確認後はコンテナを削除しましょう。  
必ず停止させてから削除してください。
```bash
$ sudo docker ps -a
$ sudo docker stop <CONTAINER ID> //コンテナを停止
$ sudo docker rm <CONTAINER ID> //コンテナを削除
$ sudo docker ps -a
```

## ECRへのプッシュ
最後にECRにプッシュを行います。

まずはECRへログインします。  
以下コマンドを実行すると、ものすごい返り値が出るのでびっくりしますが、コピペして実行しましょう。

```bash
$ aws ecr get-login --region ${REGION_NAME} --no-include-email
docker login -u AWS -p ××××××××××...(以下略)
$ sudo docker login -u AWS -p ××××××××××...(以下略)
Login Succeeded //これが出ればOK
```

続いてECR上にレポジトリを作成します。  
ECRコンソールから確認できますよ。
```bash
$ aws ecr create-repository --repository-name ${REPOSITORY_NAME} --region ${REGION_NAME}
```

返り値の *repositoryUri* を環境変数 *REPOSITORY_URI* に格納しましょう。
```bash
$ export REPOSITORY_URI=<repositoryUri>
$ export -p |grep REGISTRY_URI
```

最後にDockerイメージをECRレポジトリにプッシュします。

Dockerイメージへビルドしましょう。
```bash
sudo docker build -t ${REPOSITORY_URI}:latest .
```

ECRへプッシュします。
```bash
$ sudo docker push ${REPOSITORY_URI}:latest
```

# 確認

## マネコンから
ECRコンソールに入ってください。

リポジトリ > <YYYYMMDD>-docker-handson-repo を選択すると、
latestタグのイメージが確認できます。

## CLIから
まずaws configureにデフォルトリージョンを登録します。  
これはaws ecr list-images にリージョンを選択するオプションがないためです。
アップデートがあれば飛ばしてください。
```bash
$ aws configure
AWS Access Key ID [None]:
AWS Secret Access Key [None]:
Default region name [None]: ap-northeast-1
Default output format [None]:
```

ECRから情報を取得してきます。
```bash
$ aws ecr list-images --repository-name ${REPOSITORY_NAME}
```
JSON形式でレポジトリ内のイメージ一覧が返されるので、latestタグがあることを確認しましょう。

# 後片付け
## ECRの削除
ブログ執筆時点で、CLIから消す方法はなさそうです。
ということでマネコンから削除してください。
参考:[リポジトリの削除](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/repository-delete.html)

ECRコンソールへアクセスします。

リポジトリ > 20200701-docker-handson-repoを選択して、削除を押します。

## その他AWSリソース
以下のリソースも削除してください。
|Resource|Name|
|---|---|
|IAMRole|YYYYMMDD-20200701-docker-handson-worker-role|
|CloudFormation Stack|docker-handson-20200701|

# まとめ

前編では「ECR+ECSでwebサーバを立ち上げる」のECRの部分について触れました。  
こうゆうツールがあると自分が利用しているDockerイメージをきちんと管理できるので、仕事効率が上がって素敵ですよね。

後編では実際にプッシュしたイメージからコンテナを作成したいと思います。
なるべく早く書きます。頑張ります。

ご覧いただきありがとうございました。