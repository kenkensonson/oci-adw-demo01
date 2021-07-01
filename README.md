![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/949f5f4a-b078-f117-495f-684028ae607e.png)

# はじめに
Oracle Databaseのクラウドサービス Autonomous Databaseはあらゆるワークロードを実行できるデータベースです。OLTPやデータウェアハウスだけでなく、実は機械学習のワークロードも実行できます。

データベースで機械学習？と思われるかもしれませんが、実は技術的に理にかなった選択肢でもあります。そもそも、データベースに入っているデータはユーザーのデモグラフィックデータだったり、購買履歴だったりと、企業活動に活かせるあらゆる分析が可能なデータなのは周知のとおりです。しかもデータベースにデータが入っている時点で、正規化済だったり、時系列になっていたりするわけですから、これらを機械学習に使わない手はないわけです。

その際、データベース自体に機械学習の仕組みが備わっているのであれば、ついでに、データベースで機械学習をやってしまえばいいんじゃないの？というお話です。わざわざ別の分析システムを作ってデータベースにデータを引っ張りに行く必要もありません。しかも、プログラミングせずにGUIベースで予測モデルが作れたり、ちょっとしたMLOps的な機能までついていて、更に、巷の機械学習サービスと比べて超安価となれば試しに使ってみる価値はあると思います。

本記事ではそんな「Autonomous Database のかんたん機械学習」についてデモをベースに主要な機能を紹介します。

# デモシナリオ
「The Boston house-price data」と呼ばれる有名なデータセットを今回のデモシナリオ用に少しアレンジしたしたものを使います。このデータは米国高税調査局が収集した情報をベースにカーネギーメロン大学が作成したデータセットです。ボストンの町ごとの「犯罪率」や「非小売業の割合」など、全部で14の属性(列)を持ったデータセットになっています。今回、青色の13個の属性と赤色の1個の属性の関連性を学習し、予測モデルを構築します。すなわち、その町の13個の属性から、赤色の属性、つまり住宅価格を予測するモデルを構築するというものです。

**学習用データの構造(14属性、約470行)**

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/112e1764-8515-015c-6bc1-fe109badb0b4.png)

各列の概要は以下の通りです。

- CRIM： 犯罪率
- ZN：広い家の割合(25,000平方フィートを超える住宅地の割合)
- INDUS：非小売業の割合
- CHAS：チャールズ川隣接状況(隣接の場合：1、隣接していない場合：0）
- NOX： 一酸化窒素濃度
- RM：平均部屋数
- AGE：築古の割合(1940年より前に建てられた持ち家の割合)
- DIS：主要施設への距離(ボストン雇用センターまでの加重距離)
- RAD：主要高速道路へのアクセス性指数
- TAX：固定資産税率(10,000ドル当たり)
- PTRATIO：生徒と先生の比率
- LSTAT：低所得者人口の割合
- MEDV：住宅価格(1000ドル単位の中央値)

**予測用データの構造(13属性、30行)**
学習用データは上記のようになりますが、予測用データは当然、赤色の列(住宅価格)に値が入っていない状態のデータセットを使用します。これらのデータをロードし、最終的に予測用データの赤色の列(住宅価格)の値を予測することが本デモのシナリオになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/2bd36d77-34f4-cb02-6439-6bb5331a2223.png)

# デモの流れ
事前準備としてAutonomous Databaseのインスタンスやユーザーの作成を行います。
①学習用のデータ(CSVファイル)をデータベースにロードします。
②ロードしたデータを学習し、予測モデルを構築します。
③構築した予測モデルで、予測処理を実行します。
④予測結果のレポート用ウェブアプリを開発します
⑤、⑥、⑦でその他、ADWに付随するMLOps的な機能をいくつかご紹介します。
という流れです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/a3e6b58b-e4e7-03af-04ad-76c2a47d242b.png)

# デモの概要と動画

以降、各ステップ毎に動画と、その概説やコマンドを記載します。

### 事前準備) Autonomous DatabaseインスタンスとOMLユーザーを作成する
https://www.youtube.com/watch?v=6CtC5mgCWEM&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=2&t=1s&ab_channel=JapanOracleDevelopers

[![](https://img.youtube.com/vi/6CtC5mgCWEM&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=2&t=1s&ab_channel=JapanOracleDevelopers/0.jpg)](https://www.youtube.com/watch?v=6CtC5mgCWEM&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=2&t=1s&ab_channel=JapanOracleDevelopers)

**動画概説**
このフェーズは大きく分けて3つの作業があります。

1. Autonomous Databaseのインスタンス(以降、ADWと記載)を作成する。
何はともあれ、これをしないと何も始まりません。必要な情報を入力すれば数分でADWインスタンスが起動して利用可能状態になります。

2. 機械学習用のDBユーザーを作成する。
ADWのインスタンスを作成すると、DBユーザーとしてデフォルトではAdminのみ作成された状態になっています。Adminユーザーでは、後の工程で出てくる機械学習実行のユーザーインタフェース(AutoML UIと呼んでいます。)が利用できませんので、別途、機械学習用のユーザーを作成します。Oracle Databaseに実装されている機械学習エンジンはOracle Machine Learning(OML)と呼んでいて、このフェーズで作成する機械学習用のDBユーザーをOMLユーザーと呼びます。

3. OMLユーザーにツールの利用権限を割り当てる。
2.で作成したOMLユーザーは、後の工程で利用するデータベース・アクションというツールの利用権限がありません。そのため、下記SQLでこのユーザーにその権限を割り当てます。(下記のp_schemaには作成したOMLユーザー名、p_url_mapping_patternにはOMLユーザー名を小文字で入力します。)

```sql
BEGIN
   ORDS_ADMIN.ENABLE_SCHEMA(
     p_enAabled => TRUE,
     p_schema => 'KSONODA',
     p_url_mapping_type => 'BASE_PATH',
     p_url_mapping_pattern => 'ksonoda',
     p_auto_rest_auth => TRUE
   );
   COMMIT;
END;
```

以上、3つの作業でこのフェーズは完了です。

### Step 1) 学習用のデータを Autonomous Database にロードする
https://www.youtube.com/watch?v=V0bdw8H2qmA&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=2&ab_channel=JapanOracleDevelopers

**動画概説**
学習用のCSVファイルをAutonomous Databaseにロードし、表を作成します。(OMLではこの表を内部的にPandasのDataFrameに変換して処理を行っています。)
<ol><li>CSVファイルを[ダウンロード](https://github.com/oracle-japan/oci-adwml-demo01/archive/refs/heads/main.zip)する。
ダウンロードしたZIPファイルを解凍すると下記2つのファイルがあります。

- boston_house_prices.csv : 学習用のデータ
- boston_house_prices_new.csv : 予測用のデータ

このフェーズでは、学習用のデータ、すわなちboston_house_prices.csvのみをロードします。

</li><li>データベース・アクションからCSVファイルをデータベースにロードする。
   OMLユーザーで「データベースアクション」にログインし、データローディング機能を使って、ダウンロードした学習用データをADWのロードします。データのロードはCSVファイルをドラッグ&ドロップするだけという簡単なものです。その後、簡単なSQLでロードされたデータ(表)を確認します。
</li></ol>

以上、2つの作業でこのフェーズは完了です。

### Step 2) 予測モデルを構築する
https://www.youtube.com/watch?v=Iv9qsTOAIjI&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=3&ab_channel=JapanOracleDevelopers

**動画概説**
AutoML UIのインターフェースから、ADWにロードされたデータ(表)、予測対象の列(本シナリオでは住宅価格(MEDV))、問題のタイプ(本シナリオでは回帰(住宅価格という連続値の予測のため))を選択し、学習開始をクリックするだけです。OracleのAutoMLが下記の4つの処理を内部で自動的に処理してくれます。

<ol><li>アルゴリズムの自動選択(対応している全てのアルゴリズムを総当たりで学習し、スコアリングします。)
</li><li>特徴量の自動選択(不要な特徴量を自動的に省きます。)
</li><li>アダプティブサンプリング(精度を確保できる十分な量のデータ(行データ)を自動的にサンプリングします。)
</li><li>チューニングの自動化(各アルゴリズムがもつハイパーパラメータを自動で調整します。)</li></ol>

こられにより、現実的な時間で精度の高いモデルを構築することができます。しかも、プログラミングなしのGUIによるかんたん機械学習です。

現在の注意点としては以下2つです。
<ol><li>OMLでは非常に沢山のアルゴリズムをサポートしていますが、AutoML UI(GUIベースの機械学習)で利用できるアルゴリズムと問題のタイプは一部です。
</li><li>モデル構築はGUIベースで簡単に実行できますが、予測処理はプログラムを書く必要があります。</li></ol>

### Step 3) 予測用データをロードし、予測処理を実行する
https://www.youtube.com/watch?v=38LFwhc9zko&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=4&ab_channel=JapanOracleDevelopers


**動画概説**
構築した予測モデルを使って、実際に予測を行ってみます。

<ol>
<li>予測データをADWにロードする。</li>
Step 1でダウンロードしたファイルのうち予測用データであるboston_house_prices_new.csvをデータベースにロードします。手順は学習用データをロードしたときと全く同じです。

<li>OMLノートブックに予測用のプログラムを追加し、予測処理を実行する。</li>
構築済みの予測モデルからノートブックを生成します。そのノートブックに予測用のプログラムを追記して実行します。追記するプログラムは以下になります。
</li></ol>

```python
# 新しいデータの作成(予測用データの定義)
%python
import oml
columns ='"AGE"' , '"B"' , '"CHAS"' , '"CRIM"' , '"DIS"' , '"INDUS"' , '"LSTAT"' , '"NOX"' , '"PTRATIO"' , '"RAD"' , '"RM"' , '"TAX"' , '"ZN"'
schema='"KSONODA"'
table='"BOSTON_HOUSE_PRICES_NEW"'

column = ','.join(columns)
query = 'SELECT ' + column + ' FROM ' + schema + '.' + table

new_data = oml.sync(query=query)
z.show(new_data)

# 新しいデータで予測を実行
%python
oml_prediction = svm_mod.predict(new_data)
prediction = new_data.pull()
prediction['PREDICTION'] = oml_prediction['PREDICTION'].pull()
z.show(prediction)

# データベースに表として保存
oml.create(prediction, table = 'BOSTON_HOUSE_PRICES_PRED')
```
以上、2つの作業でこのフェーズは完了です。

### Step 4) 予測結果をレポートするためのウェブアプリケーションを開発する
https://www.youtube.com/watch?v=MLIrrwTOO6w&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=5&ab_channel=JapanOracleDevelopers

APEXを使って、予測結果をレポートし、そのレポートから簡単な分析ができるウェブアプリケーションを開発します。こちらもプログラムなしのノーコード開発の例です。

### Step 5) 予測モデルをデプロイし、アプリケーションからAPIコール(REST)できるようにする
https://www.youtube.com/watch?v=bUxFxKmyrHI&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=6&ab_channel=JapanOracleDevelopers

**動画概説**
構築済の予測モデルをデプロイし、アプリケーションからRESTで予測値をAPIコールできるようにします。今回はRESTクライアントとしてcURLコマンドを使います。cURLコマンドの実行はインターネットに接続されている環境であればどこからでも問題ありませんが、今回はOracle Cloud Shellを使います。Oracle Cloud ShellはデフォルトでcURLコマンドが使える状態になっています。まずは認証トークンを取得し、環境変数として設定後、予測元のデータをcURLコマンドでRESTエンドポイントに渡し、予測値を得ます。

<ol><li>認証トークンの取得する。
下記のコマンドで認証トークンを取得します。

```shell
curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' \
     -d '{"grant_type":"password","username":"'<OMLユーザー名>'","password":"'<OMLユーザーのパスワード>'"}' \
     https://adb.<リージョン識別子>.oraclecloud.com//omlusers/tenants/<テナンシのOCID>/databases/<データベース名>/api/oauth2/v1/token
```

- OMLユーザー名 : Step 1で作成したDBユーザー
- OMLユーザーのパスワード：Step 1で作成したDBユーザーのパスワード
- リージョン識別子：東京リージョンは「ap-tokyo-1」、大阪リージョンは「ap-osaka-1」、その他のリージョンはマニュアルの[「リージョンおよび可用性ドメインについて」](https://docs.oracle.com/ja-jp/iaas/Content/General/Concepts/regions.htm)を参照。
- テナンシのOCID : Cloud Consoleから確認できます。手順はマニュアルの[「テナンシのOCIDを確認する場所」](https://docs.oracle.com/ja-jp/iaas/Content/General/Concepts/identifiers.htm)を参照。
- データベース名：Step 1で作成したデータベースの名前

実行例

```shell
curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' \
     -d '{"grant_type":"password","username":"'KSONODA'","password":"'xxxxxxxxx'"}' \
     https://adb.ap-tokyo-1.oraclecloud.com/omlusers/tenants/ocid1.tenancy.oc1..aa・・・中略・・・tkguca/databases/KSDB01/api/oauth2/v1/token /

{"accessToken":"eyJhbGciOi・・・中略・・・vwgjQMUbz/w==","expiresIn":3600,"tokenType":"Bearer"}
```
最終行のaccessTokenの次に""で囲まれた文字列がトークンになります。

</li><li>取得した認証トークンを環境変数として設定します。

```shell
export token='eyJhbGciOi・・・中略・・・vwgjQMUbz/w=='
echo $token
```

</li><li>予測を実行する。 
予測を実行するときのcURLコマンドは下記のようなものになります。デプロイした予測モデルのRESTエンドポイントに、13個の属性値を投げると、予測値(住宅価格)を得ることができます。

```shell
curl -i -X POST --header "Authorization: Bearer ${token}" --header 'Content-Type: application/json' --header 'Accept: application/json' \
     -d '{"inputRecords": [{"CRIM":0.00632, "ZN":18, "INDUS":2.31, "CHAS":0, "NOX":0.538, "RM":6.756, "AGE":65.2, "DIS":4.09, "RAD":1, "TAX":296, "PTRATIO":15.3, "B":396.9, "LSTAT":4.98}],"topN":0, "topNdetails":0}' \
     -k https://<IPアドレス>/omlmod/v1/deployment/<URI>/score
```

- IPアドレス：デプロイしたモデルのAPI仕様の画面から確認できるIPアドレス
- URI：モデルをデプロイする際に入力する任意の文字列

実行例

```shell
curl -i -X POST --header "Authorization: Bearer ${token}" --header 'Content-Type: application/json' --header 'Accept: application/json' \
     -d '{"inputRecords": [{"CRIM":0.00632, "ZN":18, "INDUS":2.31, "CHAS":0, "NOX":0.538, "RM":6.756, "AGE":65.2, "DIS":4.09, "RAD":1, "TAX":296, "PTRATIO":15.3, "B":396.9, "LSTAT":4.98}],"topN":0, "topNdetails":0}'
     -k https://xxx.xxx.xxx.xxx/omlmod/v1/deployment/boston_svm/score
HTTP/1.1 200 OK
Date: Mon, 07 Jun 2021 08:33:02 GMT
Content-Type: application/json
Content-Length: 54
Connection: keep-alive

{"scoringResults":[{"regression":27.884171225054438}]}
```

最終行のregressionの後に表示されている値が予測値となります。


### Step 6) ノートブック(ソースコード)をバージョニングする
https://www.youtube.com/watch?v=p3nLgYfyMkE&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=7&ab_channel=JapanOracleDevelopers

ソースコードのバージョニング機能です。任意の時点でのプログラムにバージョンを付与し、編集後、どの時点のソースコードにもリストアできるようにする機能です。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/fd84b714-aa27-5203-160b-7d69f4a5a996.png)

### Step 7) ノートブックのスケジューリング実行を設定する
https://www.youtube.com/watch?v=3Kcq2dutYS4&list=PL8x2FJpi0g-tkB8X5dmNC3TribgreVwwR&index=8&ab_channel=JapanOracleDevelopers

ノートブックに書いたプログラムをスケジューリング実行する機能です。学習処理の夜間実行などをスケジューリングすることが可能です。その他、機械学習のワークフローのあらゆるフェーズを自動実行できるようにします。
