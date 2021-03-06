## はじめに
freenomでドメインを取得→Route53に登録することがあったので、備忘録です。


## 事前知識
#### NSレコード
そのドメインの管理委託先DNSサーバを定義するレコードのこと。  
ドメイン名 ns DNS権威サーバ  
ex) example.com  NS  ns-xxx.xxx.com

今回の仕組みを図で表すとおそらくこんな感じです。

[f:id:swx-sugaya:20200911231813p:plain]

## 作業手順

### ドメインの登録
[freenom](https://www.freenom.com/ja/index.html)のサイトにアクセスします。

次に、画面右上の"Sign in"から、ログインします。  
※会員登録していない人は、新規登録してください。

[f:id:swx-sugaya:20200911231819p:plain]

"Register a New Domain" を選択します。

[f:id:swx-sugaya:20200911231806p:plain]

検索バーに取得したいドメイン名を入力し、"Check Availabilty" を押します。

[f:id:swx-sugaya:20200911231831p:plain]

好きなドメインを選びましょう。今回は".tk"にします。選んだら "Get it now!"を選択します。

[f:id:swx-sugaya:20200911231825p:plain]

ステータスが "Selected" に変わったら、上部の "Checkout" を押しましょう。  
画面が切り替わったら "Continue" を押しましょう。

[f:id:swx-sugaya:20200911231837p:plain]

次の画面で、"I have read and agree to the Terms & Conditions" をチェックを付けたら、"Complete Order" を押して、購入完了です。

[f:id:swx-sugaya:20200911231843p:plain]

"MyDomains" ページから、自分の取得したドメインを確認することができます。  


### Route53のホストゾーン作成
マネコンで、Route53のページを開いてください。  
左ペインの "ホストゾーン" を選択後、"ホストゾーンの作成" を押してください。  
入力フォームが出たら、以下を参考にそれぞれを埋めて "作成"を押します。

```text
ドメイン名: freenomで取得したドメイン名
コメント: 空欄or任意
タイプ: パブリックホストゾーン
```

[f:id:swx-sugaya:20200911231849p:plain]

ホストゾーンが作成されます。  
NSレコードの値は後ほど使うので、全て控えておいてください。

### freenomドメインの委任
ここでは、Route53に取得したドメインの管理を委任するための作業を行います。  
"MyDomains" ページで、取得したドメインの "Manage Domain" を押します。

[f:id:swx-sugaya:20200911231855p:plain]

ページ遷移後、"Management Tools" タブから "Nameservers" を選択します。

[f:id:swx-sugaya:20200911231902p:plain]

"Use custom nameservers (enter below)" を選択後、控えておいたNSレコードの値を入力します。  　　
その後、"ChangeNameservers" を押しましょう。

[f:id:swx-sugaya:20200911231907p:plain]

これでドメインの管理者をfreenomから、route53のNSレコードに記載されたDNSサーバに変更しました。

## まとめ
あとは、Route53にレコード登録をどんどんしていきましょう。