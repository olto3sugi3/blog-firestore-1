#WEBアプリ開発で複数の Firebase project を使い分ける
WEBアプリを作成する際に Firebase の Firestore と Authentication は非常に使い勝手が良いので気に入ってます。
色々と使っている過程で、開発と本場で Firebase project を使い分けたり、異なる Firebase project 間でデータをコピーしたりする必要が出てきたので対応方法を考えました。
ソースは Python で作っていますが、別の言語でも同じ発想で対応できると思いますので、参考にしてください。
##Firebase へのアクセスの方法
ブラウザ（Javascript）から Firestore への直接アクセスはしていません。一般に公開するWEBアプリなので、ブラウザからのアタックが心配だったので Firestore のルールは[[ロックモード]](https://firebase.google.com/docs/firestore/quickstart?authuser=0#locked-mode)にしています。

```js
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```
クライアントからはアクセスしないので、サーバー上のPython で [[Firebase Admin SDK]](https://firebase.google.com/docs/admin/setup?hl=ja) を使うことになります。
管理者権限でのFirestore へのアクセスはサービスアカウントのキーファイルを使いますが、今回は複数のキーファイルを使い分けます。
##事前準備
###1.サービスアカウントのキーファイルを用意
複数の Firebase Project を使用するので、各々の project の数分のサービスアカウント秘密鍵を生成してJSON ファイルを保存します。
新しい秘密鍵は、Firebase のコンソールで、 プロジェクトの概要 ＝＞ プロジェクトの設定　＝＞　サービスアカウントキー　で作成できます。
作成したものは、[Firebase の プロジェクト ID].json というファイル名にして、サーバーの任意のフォルダを作って保存しておきます(私は /usr/firebase/serviceAccountKey にしました)。
###2.環境変数にフォルダ名をセット
CGI にはこのファルダ名を環境変数にセットした状態で渡します。私の環境は nginx + fcgiwrap で CGI を動かしているので、/etc/nginx/fcgiwrap.conf の一番下に以下の様にセット。

```fcgiwrap.conf
location /cgi-bin/ {
    ・・・・・・
    fastcgi_param FIREBASE_CREDENTIALS 「自分の作ったフォルダ」;
}
```
ここは、皆さんの環境（apacheなど）毎に設定方法が異なるので、ご自分の環境に合わせて設定してください。
これで準備は完了です。

##Firebase プロジェクトIDを指定すると Firebase (app) を返す Python
今回は Python で作っています。

```py
import os
import json
import firebase_admin
from firebase_admin import credentials, auth, firestore

def initapp(my_project):
    # Firebase (app) の取得
    # 既に初期化済の Firebase(app) のリストから目指すものを探す
    for myapp in firebase_admin._apps:
        if firebase_admin.get_app(myapp).project_id == my_project:
            return firebase_admin.get_app(myapp)
    # 無ければ新規に Firebase(app) を初期化する
    try:
        mycert = os.environ["FIREBASE_CREDENTIALS"] + my_project + '.json'
        cred = firebase_admin.credentials.Certificate(mycert)
        app = firebase_admin.initialize_app(cred, name=my_project) # 必ず name を指定する
        return app
    except:
        if "[DEFAULT]" in firebase_admin._apps:
            return firebase_admin.get_app()
        else:
            # Use the application default credentials = GOOGLE_APPLICATION_CREDENTIALS 
            cred = credentials.ApplicationDefault()
            default_app = firebase_admin.initialize_app(cred)
            return default_app

```
呼び出すときに、Firebase のプロジェクトIDを引数で渡します。
既にそのプロジェクトの Firebase が初期化済であれば、Firebase (app)を返します。初期化未済なら、初期化してからFirebase (app)を返します。見つからなければデフォルトのFirebaseを初期化して返します。
今回のように複数の Firebase を使い分ける際は `firebase_admin.initialize_app(cred, name=my_project)` と name を指定する必要があります。これにより、複数の Firebase Project が識別できるようになるわけです。初期化済みの Firebase を検索するときに、この name を使っています。

##使い方
###1.複数の Firebase Project を同時に使用する
異なる Firebase 間でデータをコピーするような場合は、以下のようにします

```py
# collection をコピーする
from_project = 「コピー元の Firebase プロジェクトID」
from_app = initapp(from_project)    # Firebase を初期化する
fromdb = firestore.client(from_app) # 初期化した Firebase から Firestore を取得する
to_project = 「コピー先の Firebase プロジェクトID」
to_app = initapp(to_project)        # Firebase を初期化する
todb = firestore.client(to_app)     # 初期化した Firebase から Firestore を取得する

# fromdb から todb にデータをコピーする
from_ref = fromdb.collection(「コピー元のコレクション」).document(「コピー元のドキュメントID」)
mydoc = from_ref.get()
myrec = mydoc.to_dict()
target_ref = todb.collection(「コピー先のコレクション」).document(「コピー先のドキュメントID」)
target_ref.set(myrec)
・・・・・・

```

###2.開発と本番で接続する Firebase を区別する
私はこんなマトリックスを json で作ってサーバーに保存してあります。そして、このマトリックスの保存先も環境変数として CGI に渡します。

```json
{
    "hello-world":{
        , "www.「ドメイン名」":"「本番用 Firebase プロジェクトID その１」"
        , "test.「ドメイン名」":"「テスト用 Firebase プロジェクトID その２」"
        , "test.local.「ドメイン名」":"「テスト用 Firebase プロジェクトID その２」"
    }
    , "[WEBアプリの識別名]":{
        , "www.「ドメイン名」":"「本番用 Firebase プロジェクトID その３」"
        , "test.「ドメイン名」":"「テスト用 Firebase プロジェクトID その４」"
        , "test.local.「ドメイン名」":"「テスト用 Firebase プロジェクトID その４」"
    }
    ・・・・・・

}
```
これを使って、目指す Firebase を特定して接続しに行きます。
私のサーバーでは複数のWEBアプリが動いていますが、各WEBアプリ毎に「WEBアプリの識別名」を決めています。この「WEBアプリの識別名」をリソースファイルの中に定義しておきます。

```json
{
    "app_id":「WEBアプリの識別名」
    ・・・・・・
}
```
この app_id は JavaScript から CGI を呼び出す(POST)ときに json データとして渡しています。
CGI プログラムは、マトリックスと「WEBアプリの識別名」とWEBサーバー名から FirebaseプロジェクトIDを特定して、接続しに行きます。

```py
# 「WEBアプリの識別名」とWEBサーバー名から FirebaseプロジェクトIDを特定する
myapp_id = postJson['app_id'] # JavaScript から渡された 「WEBアプリの識別名」
my_host = os.environ['SERVER_NAME'] # CGI が実行された WEB サーバー名
with open(os.environ['「マトリックスの保存先を格納した環境変数」']) as firebaseProjects:
    projectJson = json.load(firebaseProjects)
    my_project = projectJson[myapp_id][my_host]

app = initapp(my_project) # Firebase を初期化する
db = firestore.client(app) # 初期化した Firebase から Firestore を取得する

# Firestore を使った処理を実行
newdocument = db.collection("「コレクション名」").document()
・・・・・・
```
**<font color="Blue">※ このやり方のメリット</font>**
設定が面倒であれば、こんな設定は不要です。忘れてください。
私は、本番サーバーと開発サーバーとコーディング用のVSCodeで全く同じプログラムが違う環境で動くというのが気に入っています。
それよりもメリットがあるのが、同じプログラムを別のWEBアプリで使えることです。この特性を生かして、複数のWEBアプリで共通で使えそうなプログラムは部品化して共通プログラムとしてデプロイしています。
CGI を呼び出すときに「WEBアプリの識別名」を変えるだけで環境の差異を吸収してくれます。

####補足
サービスアカウントのキーファイルの保存フォルダ名やマトリックスの保存先をハードコーディングせずに環境変数で指定している理由は、サーバー（Linux）とコーディング用PC（Windows）で全く同じ Python が動くようにしたかったからです。もちろんPCには本番用のキーファイルは置いてありません。
なお、マトリックスに定義した"test.local.「ドメイン名」"というのは、コーディング用PCのVSCode の Go Live に設定しているサーバー名です。VSCode のWEBサーバーではCGIは呼び出せないのですが、各PCの環境変数 SERVER_NAME に、"test.local.「ドメイン名」"をセットしてあります。VSCode で Python をデバッグするときは、ちゃんとマトリックスに従ってテスト用のFirebaseに接続しに行きます。
さらに、コーディング（UT）用とテストサーバー（IT）で環境を分けたい場合もマトリックスに定義するだけなので簡単に分けられます。

###あとがき
WEBで検索してみましたが、まとめて解説されているサイトが見つからずに迷いました。なかなか便利なものが出来たので、ここに公開します。

こちらのwebsiteもよろしくお願いします。
[[初心者のためのWEBシステムのインフラ構築]](https://www.olto3-sugi3.tk/ja/index.html)



