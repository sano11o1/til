# Web API Good Parts Memo

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
プリフライトリクエストの対になるリクエストはシンプルリクエストとして定義されている
https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#%E5%8D%98%E7%B4%94%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88
プリフライトリクエストは OPTION メソッドを使って送信される
CORS に対応したブラウザでは状況に応じてプリフライトリクエストを自動的に行う
