# Web API Good Parts Memo

API は DB の構造と乖離していてもよい  
API はインターフェースなので、DB 側に寄りすぎるとクライアント側が使いにくくなってしまう  
どのようなユースケースで使われるのかが、API の構造を考える上で一番重要

## 件数取得のクエリ

件数を絞るクエリパラーメータは相対位置と絶対位置がある

- 相対位置の例 100 件目から 20 件を取得
- 絶対位置の例 ID: 100 から 20 件を取得

相対位置の場合、時間の経過とともに取得できる情報が変化する  
また、offset を使うとレコードの先頭から検索が走るのでパフォーマンスが悪い  
絶対位置の場合、where id > 100 のように where を使うので先頭から舐めることはなく、レコード固有の ID を使って絞っているので、時間の経過とともに情報が変化することもない

## 認証

OAuth2.0 の認可フロー(Grant Type)

- Authorization Code -> FaceBook ログインなどのサードパーティアプリケーション向け(サーバーサイド)
- Implict -> FaceBook ログインなどのサードパーティアプリケーション向け(クライアントサイド)
- Resource Owner Password Credentials -> 自社サービスの API の認証
- Client Credentials -> 認可を必要としないアプリ向け

## CORS のやりとり

CORS: 異なる生成元にアクセスするための手法

1. クライアントからサーバーへ Origin というリクエストヘッダを送る

```
Origin: http://www.example.com
```

2. サーバー側では Origin がアクセスを許可する一覧に含まれているかチェックする

3. OK だった場合は Accesss-Control-Allow-Origin レスポンスヘッダに生成元を入れて返す

```
Accesss-Control-Allow-Origin: http://www.example.com
```

アクセスしたリソースがセキュリティ上どこからでも読み込まれて良い場合は Accesss-Control-Allow-Origin に\*を入れて返す。

### プリフライトリクエスト

CORS に定義されている、生成元をまたいだリクエストを行う前にそのリクエストが受け入れられるか検証する  
プリフライトリクエストの対になるリクエストは[シンプルリクエスト](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#%E5%8D%98%E7%B4%94%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88)として定義されている  
プリフライトリクエストは OPTION メソッドを使って送信される  
CORS に対応したブラウザでは状況に応じてプリフライトリクエストを自動的に行う

## API のバージョニング

- パスパラメータに含めるのが一般的
- クエリパラメータの場合、クエリパラメータが省略された時の挙動を考慮する必要がある

# メディアタイプとセキュリティ

## XSS

JavaScript のコード を含む JSON ファイルの`Content-Type`を誤って`text/html`で配信した場合、コードが実行される場合がある

例

- ユーザーのプロフィールを登録/取得できる API がある
- プロフィールに任意の JavaScript を仕込み保存する
- 保存したプロフィールを取得するとプログラムが実行される

```
GET /users/1/profile

{
  memo: '<script>console.log('hoge')</script>'
}
```

JSON は`application/json`で配信する  
Internet Explorer には `Content Sniffing` というデータの内容からフォーマットを推測する機能がある  
この機能により`Content-Type: application/json`で配信していても、HTML と認識される可能性がある  
IE8 以降は`X-Content-Type-Options: nosniff`とすれば機能を OFF にできる  
IE7 以前は機能を OFF にできないので以下の方法で対策する

- リクエストヘッダの追加
  JSON のやりとりは URL への直接アクセスや Script の読み込みではなく、XMLHttpRequest を通じて行う。XMLHttpRequest にはヘッダを追加できる。(直接アクセスや Script の読み込みはできない)  
  任意のヘッダが設定されているかをサーバー側で検証することで、XMLHttpRequest の時のみレスポンスを返すようにすることで、Script の実行を回避する
- エスケープした結果を返すようにする

## JSON ハイジャック

前提

- ユーザー A は ソーシャルメディア facegood を使用している
- `www.facegood.com/me`にアクセスするとユーザー A のプロフィールを閲覧できる
- `www.facegood.com/me`は以下のような JSON レスポンスを返す

```
[
  'age: 21',
  'name: 'sano''
  'phoneNumber: '000-0000-0000''
]
```

悪意のある開発者が運営するサイト(`www.hackyou`)には以下のような Script が仕掛けられている
script タグは同一生成元ポリシーの対象外のため、ユーザー A が hackyou に訪れた際、`'www.facegood.com/me`へリクエストが飛ぶ。

```
<script src='www.facegood.com/me'>
```

上記のレスポンスがダウンロードされるのは、ユーザー A のブラウザに限定されるので、悪意のある開発者はレスポンスの結果を取得できない  
JSON ハイジャックを使うと、ダウンロードしたページがレスポンスにアクセスできてしまう

Array オブジェクトのコンストラクタを変更する JSON ハイジャックの例

配列として渡された JSON は JavaScript の構文として読み取れるためページ内で操作が可能だった(バグは解消済み)

### 対策

- JSON を SCRIPT 要素では読み込めないようにする(XSS のヘッダーの追加と同じ対応)
- JSON のレスポンスをブラウザが必ず JSON と認識するようにする
  - Content-Type 属性に application/json を指定
- JSON を JavaScript として実行しないようにする
  - 配列ではなくオブジェクトを返すようにすれば、仮に Javascript として読み込まれても構文エラーになる
