---
title: "HTTP Headerの備忘録"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["HTTP"]
published: true
---

## 🧩 HTTP ヘッダーとは

HTTP ヘッダーとは、  
**クライアント（ブラウザなど）とサーバーが通信を行う際に、  
リクエストやレスポンスのメタ情報をやり取りする部分**。

たとえば：

- どんな形式のデータを欲しいか (`Accept`)
- どんなブラウザから来ているか (`User-Agent`)
- 認証情報 (`Authorization`)
- Cookie やキャッシュの制御 など

---

## 📤 リクエストヘッダーの主な項目一覧

| ヘッダー名            | 説明                                                                           | 例                                                                                                                            |
| --------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| **Accept**            | クライアントが受け入れ可能なメディアタイプ（コンテンツの形式）を指定する。     | `Accept: text/html, application/json` → HTML か JSON を受け取れる。                                                           |
| **Accept-Charset**    | 受け入れ可能な文字セット（エンコーディング）を指定。                           | `Accept-Charset: UTF-8`                                                                                                       |
| **Accept-Encoding**   | 受け入れ可能な圧縮方式を指定。サーバーは可能ならその形式で圧縮して返す。       | `Accept-Encoding: gzip, deflate`                                                                                              |
| **Accept-Language**   | 受け入れ可能な言語を指定。                                                     | `Accept-Language: ja,en-US;q=0.9` → 日本語優先、英語も可。                                                                    |
| **Authorization**     | 認証情報を送る。Basic 認証や Bearer トークンなど。                             | `Authorization: Basic dXNlcjpwYXNz`（user:pass を Base64）<br>`Authorization: Bearer eyJhbGciOiJIUzI1NiIs...`（JWT トークン） |
| **Cache-Control**     | キャッシュの制御。クライアントやプロキシがどのようにキャッシュを扱うかを指示。 | `Cache-Control: no-cache`（キャッシュを使わず再取得）                                                                         |
| **Connection**        | 通信の持続や切断を制御。                                                       | `Connection: keep-alive`（次のリクエストでも同じ接続を再利用）                                                                |
| **Content-Length**    | リクエストボディのバイト数。POST や PUT のときに使用。                         | `Content-Length: 348`                                                                                                         |
| **Content-Type**      | ボディのデータ形式を指定。                                                     | `Content-Type: application/json`、`Content-Type: multipart/form-data`                                                         |
| **Cookie**            | ブラウザに保存されている Cookie 情報をサーバーへ送信。                         | `Cookie: session_id=abc123; theme=dark`                                                                                       |
| **Date**              | リクエスト送信時の日時。                                                       | `Date: Wed, 22 Oct 2025 09:00:00 GMT`                                                                                         |
| **Expect**            | サーバーに特定の動作を要求。                                                   | `Expect: 100-continue`                                                                                                        |
| **From**              | リクエストを送っているユーザーのメールアドレス。一般的にはあまり使われない。   | `From: user@example.com`                                                                                                      |
| **Host**              | 要求しているホスト名とポート番号。HTTP/1.1 では必須。                          | `Host: example.com`                                                                                                           |
| **If-Match**          | 条件付きリクエスト。ETag が一致する場合のみ実行。                              | `If-Match: "abc123etag"`                                                                                                      |
| **If-Modified-Since** | 指定日時以降に変更があった場合のみ取得。キャッシュの節約に使う。               | `If-Modified-Since: Tue, 21 Oct 2025 10:00:00 GMT`                                                                            |
| **If-None-Match**     | ETag が一致しない場合のみ取得。キャッシュ確認によく使う。                      | `If-None-Match: "xyz789etag"`                                                                                                 |
| **Origin**            | リクエスト元のオリジン（スキーム＋ドメイン＋ポート）。CORS で重要。            | `Origin: https://myapp.com`                                                                                                   |
| **Range**             | ファイルの一部だけを要求（部分ダウンロードなど）。                             | `Range: bytes=0-499`（最初の 500 バイトだけ）                                                                                 |
| **Referer**           | リクエスト元のページ URL。                                                     | `Referer: https://google.com/search?q=example`                                                                                |
| **TE**                | 受け入れ可能な転送エンコーディングを指定。                                     | `TE: trailers`                                                                                                                |
| **User-Agent**        | クライアントソフト（ブラウザやアプリ）の情報。                                 | `User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...`                                                              |

---

## 📥 レスポンスヘッダーの主な項目

| ヘッダー名                      | 説明                                                       | 例                                                       |
| ------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------- |
| **Content-Type**                | 返すデータの形式を示す。                                   | `Content-Type: text/html; charset=UTF-8`                 |
| **Content-Length**              | レスポンスボディのサイズ（バイト数）。                     | `Content-Length: 1045`                                   |
| **Content-Encoding**            | データを圧縮している場合、その方式を示す。                 | `Content-Encoding: gzip`                                 |
| **Cache-Control**               | クライアントやプロキシへのキャッシュ指示。                 | `Cache-Control: max-age=3600`（1 時間キャッシュ OK）     |
| **Set-Cookie**                  | クッキーをクライアントに保存させる指示。                   | `Set-Cookie: session_id=abc123; HttpOnly; Secure`        |
| **Location**                    | リダイレクト先 URL。                                       | `Location: https://example.com/login`                    |
| **Server**                      | サーバーソフトウェア名。                                   | `Server: nginx/1.23.4`                                   |
| **Date**                        | レスポンス生成日時。                                       | `Date: Wed, 22 Oct 2025 09:05:00 GMT`                    |
| **ETag**                        | リソースのバージョン識別子。キャッシュ制御で使う。         | `ETag: "abc123etag"`                                     |
| **Access-Control-Allow-Origin** | CORS で許可するオリジン。                                  | `Access-Control-Allow-Origin: *`（全て許可）             |
| **WWW-Authenticate**            | 認証が必要なとき、認証方式を指定。                         | `WWW-Authenticate: Basic realm="User Area"`              |
| **Retry-After**                 | 次のリクエストまで待つ秒数や日時を指定（503 などで使用）。 | `Retry-After: 120`（120 秒後に再試行）                   |
| **Content-Disposition**         | ファイルをダウンロードさせる場合の指定。                   | `Content-Disposition: attachment; filename="report.pdf"` |

---

## 🔒 よく使われるセキュリティ関連ヘッダー

| ヘッダー名                           | 目的                                                       | 例                                                               |
| ------------------------------------ | ---------------------------------------------------------- | ---------------------------------------------------------------- |
| **Strict-Transport-Security (HSTS)** | HTTPS 接続を強制する。                                     | `Strict-Transport-Security: max-age=31536000; includeSubDomains` |
| **X-Frame-Options**                  | 他サイトからの iframe 埋め込みを防止。                     | `X-Frame-Options: SAMEORIGIN`                                    |
| **X-Content-Type-Options**           | MIME タイプの自動判定を防止。                              | `X-Content-Type-Options: nosniff`                                |
| **Referrer-Policy**                  | Referer ヘッダーの送信範囲を制御。                         | `Referrer-Policy: no-referrer-when-downgrade`                    |
| **Content-Security-Policy (CSP)**    | 外部スクリプトやリソースの読み込み制御。XSS 防止に効果的。 | `Content-Security-Policy: default-src 'self'; img-src https:`    |

---

## 🧠 まとめ

- HTTP ヘッダーは「通信時の設定書」みたいなもの。
- リクエスト側は「どんな情報を欲しいか・誰なのか」を伝える。
- レスポンス側は「どんなデータを返したか・どう扱ってほしいか」を示す。
- 適切に設定することで、安全・効率的な通信ができるようになる 💡
