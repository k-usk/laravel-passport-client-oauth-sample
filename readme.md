# Larave Passport サンプル
## 趣旨

Laravel Passportを使ってAPIからのアクセスにOAuth認証をかけたい。  
一般に公開するものではなく、特定のクライアントからのアクセスのみを想定する。  
そのため、ユーザデータは使用しない。

## インストール
Laravel Passportをインストール。

```
$ composer require laravel/passport
```

Package Auto-Discovery に対応しているため、プロバイダに追加してやる作業は必要ない。

* [Laravel Package Auto-Discovery - Laravel News](https://laravel-news.com/package-auto-discovery)

## セットアップ
DBに必要なテーブルを作成する。

```
$ php artisan migrate
```

OAuth用に以下のテーブルが作成される。

* `oauth_access_tokens`
* `oauth_auth_codes`
* `oauth_clients`
* `oauth_personal_access_clients`
* `oauth_refresh_tokens`

トークン作成時に使用されるキーを生成する。

```
$ php artisan passport:install
```

キーは、`/storage/`以下に生成される。  
デフォルトでは`.gitignore`で無視する設定となっているので注意。  
また、公開リポジトリにアップしてはいけない。  
対処法などは以下参考。

* [Laravel Passport keyファイルの扱い - Qiita](https://qiita.com/kawax/items/59fde47056816cec52ec)

また、キーの生成とともに、DBにクライアントが作成される。  

```
Encryption keys generated successfully.
Personal access client created successfully.
Client ID: 1
Client Secret: tR7FSAHLQ8qw1xIgEWMKQ26QK2nKUxHahSHvY3RW
Password grant client created successfully.
Client ID: 2
Client Secret: 6n4TGzdrJHYdEJwPoMsaYuCA9EaFpXiGFr4dMVc8
```

一つ目が、 `Laravel Personal Access Client`  
二つ目が、`Laravel Password Grant Client`

二つ目はユーザ名＋パスワードを利用したアクセストークンの発行に利用出来る。  
(ユーザーとの紐付けは特に必要なし)  
(必要なければ消しておいてもよい)

### コードに追加
#### `AuthServiceProvider.php`に追加
`/app/Providers/AuthServiceProvider.php`

```php
public function boot()
{
    $this->registerPolicies();

    Passport::routes();
}
```

#### `auth.php`のdriverをpassportに変更
`/config/auth.php`

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
```

## クライアント認証情報グラントトークン
冒頭の趣旨を実現するために、マシン-マシン間の認証に最適、という認証方式を採用する。

#### `Kanel.php`に追加
`/app/Http/Kanel.php`

```php
protected $routeMiddleware = [
	...
	'client' => CheckClientCredentials::class,
];	
```

## アクセストークンの取得

以下にアクセスして取得出来る。

・リクエスト

```
POST : /oauth/token
```

| 項目 | 内容 |
| :-- | :-- |
| `grant_type` | `client_credentials` |
| `client_id` | 発行したクライアントのID(数字) |
| `client_secret` | 発行したクライアントのシークレット |
| `scope` | アクセスするスコープ |

・レスポンス

```
{
	"token_type": "Bearer",
	"expires_in": 31536000,
	"access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImp0aSI6ImJiMmFkYWQyMTNiMDYxZGM1N2Q3NzFiOGY4MGQxZDYwOWVhYTgwODc4MzQzYTRkOTQ4NDM0OGNjZDEyMzM1MWU3NzM0NTZjZjAzMzE2NDliIn0.eyJhdWQiOiIxIiwianRpIjoiYmIyYWRhZDIxM2IwNjFkYzU3ZDc3MWI4ZjgwZDFkNjA5ZWFhODA4NzgzNDNhNGQ5NDg0MzQ4Y2NkMTIzMzUxZTc3MzQ1NmNmMDMzMTY0OWIiLCJpYXQiOjE1MDg0NzIxOTIsIm5iZiI6MTUwODQ3MjE5MiwiZXhwIjoxNTQwMDA4MTkyLCJzdWIiOiIiLCJzY29wZXMiOltdfQ.qaizE8P9cs-tYBm_XD5IJOmrV4NxgSLkMBHj5nj5AKvE2RYfGbXjwv_dvuk_V2ObXJ8kmcidbfnWA_UHmpoXWrG01twouBHZMy8iSuSdo9ymo7LzNnHKCqLBotj75Vhp2p8ASEPaGhiqWZ8jCRJGFaWfJfsQLZeDIyhAz6N1xegaENHkeGUh4rojmK5rNc85T2kI_Ec__5qo3Yu94QnqBxqvDPWLEHzGQ0Qi3YZyNudS11fvVBZNtvlc9iF65rFGqPBVuDGCJEHbkO8pf8FlRXOFYOAXAtl5SEHJdoroG_5PcZIbZ6Ej_ssF-Ns-n6Ldsd0NOqxPE-Q45WY2FWDM3vkvbg6QF3b_zclwyGJvg5rl54lXr67nXEyXagVIbVTTPtQ-iZJReYN1SKc8YlQ1lh5EaNNhZ10seuIeZzhcetKq5qaGBbFAO9lvXg4JshybJ-hUgIVHBRkLySJ6HTYkMjh2gSEc_n3AHGCXqOBUbPwM0ZWYz4k00eqeGu_K1CowB1srrgecbLl4Vi2QYjtfr1hEJuu4cF_bole2MV7ts-2g2egt6qCKE4Au33agHBAvxCrFM631C_d1Se_Bq2QAp2erOKInaltyOSMsnOmjvoRLscYZ7zu1tWgjdDq_aCWaUUuiBal_n34rB4y73OP2tMh963KG54VFyoKMeqYW4hA"
}
```

発行されたトークンは、`oauth_access_tokens`テーブルに格納されていく。

### 有効期限
デフォルトではトークンの有効期限は１年間となっている。

変更するには、`AuthServiceProvider`の`boot`メソッドから変更可能。

`/app/Providers/AuthServiceProvider.php`

```php
public function boot()
{
    $this->registerPolicies();

    Passport::routes();
    Passport::tokensExpireIn(Carbon::now()->addMinute(60));
    Passport::refreshTokensExpireIn(Carbon::now()->addHour(2));
}
```

## APIへのアクセス
上記トークンを使用してアクセス制限をかけたAPIへアクセスする。

### ルートに追加
追加したミドルウェアを使用して、apiのルートにこの認証で使用するエンドポイントのリクエストを追加。  

`/routes/api.php`

```php
Route::get('/hoge', function (){
    return 'OK';
})->middleware('client');
```

### アクセス
apiのルーティングは、`/api`以下に作成される。  
よって、エンドポイントは以下になる

```
GET : /api/hoge
```

ヘッダにアクセストークンを付与してアクセスする。

* Authorization : `Bearer YOUR-ACCESS-TOKEN`

#### エラー
アクセストークンが間違っているなどの場合は、`InvalidArgumentException`が発生する。  

# 参考

* [API認証(Passport) 5.5 Laravel](https://readouble.com/laravel/5.5/ja/passport.html)
* [Laravel5.5でAPI認証のパッケージ(Laravel Passport)を利用する - Qiita](https://qiita.com/niiyz/items/fffff94acb6061ecc9d4)