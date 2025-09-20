# REST API設計の基礎と実践

> **対象者**: フロントエンドエンジニア → バックエンド学習者  
> **学習目標**: 実践的なREST API設計スキルの習得  
> **作成日**: 2025-09-15

## 📚 目次

- [[#1. REST APIとは何か|1. REST APIとは何か]]
- [[#2. HTTP メソッドとステータスコード|2. HTTP メソッドとステータスコード]]
- [[#3. URL設計の原則|3. URL設計の原則]]
- [[#4. リクエスト・レスポンス設計|4. リクエスト・レスポンス設計]]
- [[#5. エラーハンドリング|5. エラーハンドリング]]
- [[#6. 認証・認可|6. 認証・認可]]
- [[#7. 実装例（Node.js/Express）|7. 実装例（Node.js/Express）]]
- [[#8. パフォーマンス最適化|8. パフォーマンス最適化]]
- [[#9. 参考記事・リソース|9. 参考記事・リソース]]

---

## 1. REST APIとは何か

### RESTの6原則

**REST**（Representational State Transfer）は、Web APIの設計スタイルです。

1. **クライアント・サーバー分離**
   - フロントエンドとバックエンドの独立性
   - 各々が独立して進化可能

2. **ステートレス**
   ```javascript
   // ❌ 悪い例：サーバーにセッション状態を保持
   GET /api/next-page
   
   // ✅ 良い例：リクエストに必要な情報を含める
   GET /api/users?page=2&limit=10
   ```

3. **キャッシュ可能**
   - レスポンスヘッダーでキャッシュ戦略を明示

4. **統一インターフェース**
   - 一貫したURL構造とHTTPメソッドの使用

5. **階層システム**
   - ロードバランサー、プロキシの透過的な利用

6. **コードオンデマンド**（オプション）
   - 必要に応じてクライアントにコードを送信

### フロントエンドエンジニアの視点

フロントエンドでこんな経験ありませんか？

```javascript
// 😤 「なんでこのAPIは統一感がないんだ...」
fetch('/api/getUsers')     // GET なのに get を入れる
fetch('/api/user-delete')  // DELETE メソッドを使わない
fetch('/api/products', {   // POST なのにデータ取得
  method: 'POST'
})
```

良いREST API設計を学ぶことで、このような問題を解決できます。

---

## 2. HTTP メソッドとステータスコード

### 主要なHTTPメソッド

| メソッド | 用途 | 冪等性 | 安全性 |
|---------|------|--------|-------|
| GET | リソース取得 | ✅ | ✅ |
| POST | リソース作成 | ❌ | ❌ |
| PUT | リソース更新/作成 | ✅ | ❌ |
| PATCH | リソース部分更新 | ❌ | ❌ |
| DELETE | リソース削除 | ✅ | ❌ |

### 実践的な使い分け

```javascript
// ユーザー管理API の設計例

// ✅ 適切なメソッド使用
GET    /api/users           // ユーザー一覧取得
GET    /api/users/123       // 特定ユーザー取得
POST   /api/users           // 新規ユーザー作成
PUT    /api/users/123       // ユーザー全体更新
PATCH  /api/users/123       // ユーザー部分更新
DELETE /api/users/123       // ユーザー削除
```

### 重要なステータスコード

#### 成功系（2xx）
- **200 OK**: 成功（GET, PUT, PATCH）
- **201 Created**: リソース作成成功（POST）
- **204 No Content**: 成功、レスポンスボディなし（DELETE）

#### クライアントエラー（4xx）
- **400 Bad Request**: リクエスト形式エラー
- **401 Unauthorized**: 認証が必要
- **403 Forbidden**: 権限不足
- **404 Not Found**: リソースが存在しない
- **422 Unprocessable Entity**: バリデーションエラー

#### サーバーエラー（5xx）
- **500 Internal Server Error**: サーバー内部エラー
- **503 Service Unavailable**: サービス利用不可

---

## 3. URL設計の原則

### 基本的な設計原則

1. **名詞を使用、動詞は避ける**
   ```javascript
   // ❌ 悪い例
   GET /api/getUsers
   POST /api/createUser
   DELETE /api/deleteUser/123
   
   // ✅ 良い例
   GET /api/users
   POST /api/users
   DELETE /api/users/123
   ```

2. **複数形を使用**
   ```javascript
   // ✅ 推奨
   GET /api/users
   GET /api/products
   
   // ❌ 非推奨（一貫性がない）
   GET /api/user
   GET /api/product
   ```

3. **階層構造を表現**
   ```javascript
   // ユーザーの投稿を取得
   GET /api/users/123/posts
   
   // 特定の投稿のコメントを取得
   GET /api/posts/456/comments
   
   // 特定のコメントを取得
   GET /api/posts/456/comments/789
   ```

### クエリパラメータの活用

```javascript
// フィルタリング
GET /api/users?status=active&role=admin

// ソート
GET /api/products?sort=price&order=desc

// ページネーション
GET /api/posts?page=2&limit=20

// フィールド選択
GET /api/users?fields=id,name,email
```

### 実際のAPI設計例

```javascript
// ECサイトのAPI設計
GET    /api/products                    // 商品一覧
GET    /api/products/123                // 特定商品詳細
GET    /api/products?category=electronics&price_min=1000
POST   /api/products                    // 商品作成
PUT    /api/products/123                // 商品更新

GET    /api/products/123/reviews        // 商品のレビュー一覧
POST   /api/products/123/reviews        // レビュー投稿
GET    /api/reviews/456                 // 特定レビュー詳細

GET    /api/users/789/orders            // ユーザーの注文履歴
GET    /api/orders/321                  // 特定注文詳細
POST   /api/orders                      // 注文作成
```

---

## 4. リクエスト・レスポンス設計

### リクエストボディ設計

```javascript
// ✅ 良いリクエスト設計
POST /api/users
{
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "age": 25,
  "profile": {
    "bio": "フロントエンドエンジニア",
    "location": "東京"
  }
}

// ✅ 部分更新の例
PATCH /api/users/123
{
  "profile": {
    "bio": "フルスタックエンジニア"
  }
}
```

### レスポンス設計

#### 成功レスポンス
```javascript
// 単一リソース
GET /api/users/123
{
  "data": {
    "id": 123,
    "name": "田中太郎",
    "email": "tanaka@example.com",
    "created_at": "2025-09-15T10:00:00Z",
    "updated_at": "2025-09-15T10:00:00Z"
  }
}

// コレクション
GET /api/users
{
  "data": [
    {
      "id": 123,
      "name": "田中太郎",
      "email": "tanaka@example.com"
    }
  ],
  "meta": {
    "total": 1250,
    "page": 1,
    "per_page": 20,
    "total_pages": 63
  },
  "links": {
    "first": "/api/users?page=1",
    "last": "/api/users?page=63",
    "next": "/api/users?page=2",
    "prev": null
  }
}
```

#### エラーレスポンス
```javascript
// バリデーションエラー
HTTP 422 Unprocessable Entity
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に問題があります",
    "details": [
      {
        "field": "email",
        "message": "有効なメールアドレスを入力してください"
      },
      {
        "field": "age",
        "message": "年齢は18歳以上である必要があります"
      }
    ]
  }
}
```

---

## 5. エラーハンドリング

### エラーレスポンスの統一フォーマット

```javascript
// 標準エラーフォーマット
{
  "error": {
    "code": "ERROR_CODE",
    "message": "ユーザー向けメッセージ",
    "details": "開発者向け詳細情報（任意）",
    "trace_id": "req_123456789" // デバッグ用
  }
}
```

### エラーコードの体系化

```javascript
// エラーコード例
const ERROR_CODES = {
  // 認証関連
  AUTH_REQUIRED: 'AUTH_001',
  INVALID_TOKEN: 'AUTH_002',
  TOKEN_EXPIRED: 'AUTH_003',
  
  // バリデーション関連
  VALIDATION_ERROR: 'VAL_001',
  REQUIRED_FIELD: 'VAL_002',
  INVALID_FORMAT: 'VAL_003',
  
  // リソース関連
  RESOURCE_NOT_FOUND: 'RES_001',
  RESOURCE_CONFLICT: 'RES_002',
  
  // ビジネスロジック関連
  INSUFFICIENT_BALANCE: 'BIZ_001',
  ORDER_ALREADY_SHIPPED: 'BIZ_002'
};
```

### フロントエンドでの活用例

```javascript
// フロントエンドでのエラーハンドリング
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    const data = await response.json();
    
    if (!response.ok) {
      switch (data.error.code) {
        case 'AUTH_001':
          // ログインページにリダイレクト
          redirectToLogin();
          break;
        case 'RES_001':
          // ユーザーが見つからない
          showNotFoundMessage();
          break;
        default:
          // 汎用エラー
          showGenericError(data.error.message);
      }
      return null;
    }
    
    return data.data;
  } catch (error) {
    // ネットワークエラーなど
    showNetworkError();
    return null;
  }
}
```

---

## 6. 認証・認可

### JWT（JSON Web Token）を使った認証

```javascript
// ログイン
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// レスポンス
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 3600,
    "user": {
      "id": 123,
      "name": "田中太郎",
      "email": "user@example.com"
    }
  }
}
```

### 認証が必要なAPIの呼び出し

```javascript
// Authorization ヘッダーでトークン送信
fetch('/api/protected-resource', {
  headers: {
    'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
    'Content-Type': 'application/json'
  }
});
```

### リフレッシュトークン

```javascript
// トークンリフレッシュ
POST /api/auth/refresh
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

// レスポンス
{
  "data": {
    "access_token": "new_access_token",
    "expires_in": 3600
  }
}
```

---

## 7. 実装例（Node.js/Express）

### 基本的なサーバー構成

```javascript
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

const app = express();

// ミドルウェア
app.use(helmet()); // セキュリティヘッダー
app.use(cors()); // CORS対応
app.use(express.json({ limit: '10mb' })); // JSON パース

// レート制限
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100 // 最大100リクエスト
});
app.use('/api/', limiter);

// ルート定義
app.use('/api/users', require('./routes/users'));
app.use('/api/posts', require('./routes/posts'));

// エラーハンドリングミドルウェア
app.use((err, req, res, next) => {
  console.error(err.stack);
  
  if (err.type === 'ValidationError') {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: '入力内容に問題があります',
        details: err.details
      }
    });
  }
  
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'サーバー内部エラーが発生しました'
    }
  });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### ユーザーリソースの実装例

```javascript
// routes/users.js
const express = require('express');
const { body, validationResult, param } = require('express-validator');
const router = express.Router();

// バリデーションルール
const userValidation = [
  body('name').notEmpty().withMessage('名前は必須です'),
  body('email').isEmail().withMessage('有効なメールアドレスを入力してください'),
  body('age').isInt({ min: 18 }).withMessage('年齢は18歳以上である必要があります')
];

// ユーザー一覧取得
router.get('/', async (req, res) => {
  try {
    const { page = 1, limit = 20, search, status } = req.query;
    
    // クエリ構築（実際にはDBライブラリを使用）
    let query = 'SELECT * FROM users WHERE 1=1';
    const params = [];
    
    if (search) {
      query += ' AND (name LIKE ? OR email LIKE ?)';
      params.push(`%${search}%`, `%${search}%`);
    }
    
    if (status) {
      query += ' AND status = ?';
      params.push(status);
    }
    
    query += ' ORDER BY created_at DESC LIMIT ? OFFSET ?';
    params.push(parseInt(limit), (parseInt(page) - 1) * parseInt(limit));
    
    // DB実行（疑似コード）
    const users = await db.query(query, params);
    const total = await db.query('SELECT COUNT(*) as count FROM users')[0].count;
    
    res.json({
      data: users,
      meta: {
        total,
        page: parseInt(page),
        per_page: parseInt(limit),
        total_pages: Math.ceil(total / limit)
      }
    });
  } catch (error) {
    next(error);
  }
});

// 特定ユーザー取得
router.get('/:id', 
  param('id').isInt().withMessage('有効なユーザーIDを指定してください'),
  async (req, res, next) => {
    try {
      const errors = validationResult(req);
      if (!errors.isEmpty()) {
        return res.status(400).json({
          error: {
            code: 'VALIDATION_ERROR',
            message: 'パラメータが不正です',
            details: errors.array()
          }
        });
      }
      
      const user = await db.query('SELECT * FROM users WHERE id = ?', [req.params.id]);
      
      if (user.length === 0) {
        return res.status(404).json({
          error: {
            code: 'USER_NOT_FOUND',
            message: 'ユーザーが見つかりません'
          }
        });
      }
      
      res.json({
        data: user[0]
      });
    } catch (error) {
      next(error);
    }
  }
);

// ユーザー作成
router.post('/', userValidation, async (req, res, next) => {
  try {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(422).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: '入力内容に問題があります',
          details: errors.array()
        }
      });
    }
    
    const { name, email, age } = req.body;
    
    // メール重複チェック
    const existingUser = await db.query('SELECT id FROM users WHERE email = ?', [email]);
    if (existingUser.length > 0) {
      return res.status(409).json({
        error: {
          code: 'EMAIL_ALREADY_EXISTS',
          message: 'このメールアドレスは既に使用されています'
        }
      });
    }
    
    // ユーザー作成
    const result = await db.query(
      'INSERT INTO users (name, email, age, created_at) VALUES (?, ?, ?, NOW())',
      [name, email, age]
    );
    
    const newUser = await db.query('SELECT * FROM users WHERE id = ?', [result.insertId]);
    
    res.status(201).json({
      data: newUser[0]
    });
  } catch (error) {
    next(error);
  }
});

module.exports = router;
```

---

## 8. パフォーマンス最適化

### キャッシュ戦略

```javascript
// レスポンスヘッダーでキャッシュ制御
app.get('/api/users/:id', async (req, res) => {
  const user = await getUserById(req.params.id);
  
  // 5分間キャッシュ
  res.set({
    'Cache-Control': 'public, max-age=300',
    'ETag': generateETag(user),
    'Last-Modified': user.updated_at
  });
  
  res.json({ data: user });
});

// ETags を使った条件付きリクエスト
app.get('/api/users/:id', async (req, res) => {
  const user = await getUserById(req.params.id);
  const etag = generateETag(user);
  
  // クライアントのETagと比較
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end(); // Not Modified
  }
  
  res.set('ETag', etag);
  res.json({ data: user });
});
```

### ページネーション戦略

```javascript
// オフセットベース（従来型）
GET /api/posts?page=2&limit=20

// カーソルベース（パフォーマンス重視）
GET /api/posts?cursor=eyJpZCI6MTIzfQ&limit=20

// 実装例
router.get('/posts', async (req, res) => {
  const { cursor, limit = 20 } = req.query;
  
  let query = 'SELECT * FROM posts';
  const params = [];
  
  if (cursor) {
    // カーソルをデコードしてIDを取得
    const { id } = JSON.parse(Buffer.from(cursor, 'base64').toString());
    query += ' WHERE id < ?';
    params.push(id);
  }
  
  query += ' ORDER BY id DESC LIMIT ?';
  params.push(parseInt(limit) + 1); // +1 for hasMore判定
  
  const posts = await db.query(query, params);
  const hasMore = posts.length > limit;
  
  if (hasMore) {
    posts.pop(); // 余分な1件を削除
  }
  
  const nextCursor = hasMore 
    ? Buffer.from(JSON.stringify({ id: posts[posts.length - 1].id })).toString('base64')
    : null;
  
  res.json({
    data: posts,
    meta: {
      has_more: hasMore,
      next_cursor: nextCursor
    }
  });
});
```

### レスポンス圧縮

```javascript
const compression = require('compression');

app.use(compression({
  level: 6, // 圧縮レベル
  threshold: 1024, // 1KB以上で圧縮
  filter: (req, res) => {
    // 画像ファイルは圧縮しない
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  }
}));
```

---

## 9. 参考記事・リソース

### 📖 必読記事

#### 日本語記事
1. **[RESTful APIとは何なのか](https://qiita.com/NagaokaKenichi/items/0647c30ef596cedf4bf2)** - Qiita
   - REST APIの基本概念を分かりやすく解説

2. **[Web API設計のベストプラクティス](https://zenn.dev/yodaka/articles/web-api-design-best-practices)** - Zenn
   - 実践的なAPI設計のガイドライン

3. **[HTTPステータスコードを適切に使い分けよう](https://qiita.com/uenosy/items/ba9dbc70781bddc4a491)** - Qiita
   - ステータスコードの正しい使い方

4. **[JWT認証の仕組みとNode.jsでの実装](https://zenn.dev/hsaki/articles/jwt-auth-nodejs)** - Zenn
   - JWT認証の実装方法

#### 英語記事（重要）
5. **[REST API Tutorial](https://restfulapi.net/)** 
   - REST APIの包括的なガイド

6. **[Microsoft REST API Guidelines](https://github.com/Microsoft/api-guidelines)**
   - 大企業のAPI設計ガイドライン

7. **[Google API Design Guide](https://cloud.google.com/apis/design)**
   - Googleが推奨するAPI設計原則

### 🛠 実装参考

#### Node.js/Express
8. **[Express.js公式ドキュメント](https://expressjs.com/)**

9. **[Node.js REST API Tutorial](https://zenn.dev/yukiji/articles/nodejs-express-rest-api)** - Zenn

#### バリデーション
10. **[express-validator使い方](https://qiita.com/sin_tanaka/items/ea149a33bd3e5b624b5d)** - Qiita

#### セキュリティ
11. **[Node.js Security Best Practices](https://blog.risingstack.com/node-js-security-checklist/)**

12. **[Web APIのセキュリティ対策](https://zenn.dev/dove/articles/web-api-security)** - Zenn

### 📊 ツール・ライブラリ

#### API設計・ドキュメント
- **[Swagger/OpenAPI](https://swagger.io/)** - API仕様書作成
- **[Postman](https://www.postman.com/)** - API テスト
- **[Insomnia](https://insomnia.rest/)** - API クライアント

#### Node.js関連
- **[express-rate-limit](https://www.npmjs.com/package/express-rate-limit)** - レート制限
- **[helmet](https://www.npmjs.com/package/helmet)** - セキュリティヘッダー
- **[joi](https://www.npmjs.com/package/joi)** - バリデーション
- **[express-validator](https://www.npmjs.com/package/express-validator)** - リクエストバリデーション

### 🚀 最新技術記事（2025年版）

#### 最新のAPI設計トレンド
13. **[REST API設計の実践 – ベストプラクティスとその落とし穴](https://speakerdeck.com/kentaroutakeda/rest-apishe-ji-noshi-jian-besutopurakuteisutosonoluo-tosixue)** - Postman API Night Tokyo 2025 Spring
    - 2025年5月の最新発表資料、実際のケーススタディを含む

14. **[「なんとなく」でやらないための私的Web API設計ノウハウ](https://zenn.dev/arsaga/articles/4a72774b1c93d2)** - Zenn
    - 実務での判断基準と根拠に基づいた設計アプローチ

15. **[REST API 設計指針・セキュリティ編](https://zenn.dev/kameoncloud/articles/2c739be7c5aaf5)** - Zenn 2025年4月
    - OpenAPI 3.1を使ったセキュリティ対策の最新動向

#### 実践的なNode.js/Express記事
16. **[【Node.js】Express.jsを使う場合と使わない場合でWeb API比較](https://dev.classmethod.jp/articles/express-js-web-api/)** - クラスメソッド 2024年9月
    - Express.jsの利点を具体的なコード比較で解説

17. **[Express実践入門](https://gist.github.com/mitsuruog/fc48397a8e80f051a145)** - GitHub
    - 実際のプロジェクト構成とベストプラクティス

### 📚 学習リソース

#### 書籍
18. **「Web API: The Good Parts」** - 水野貴明
    - 日本語で学べるWeb API設計の決定版

19. **「RESTful Web APIs」** - Leonard Richardson
    - REST APIの深い理解のための英語書籍

#### 公式ドキュメント・チュートリアル
20. **[Express.js 公式サイト](https://expressjs.com/ja/)** 
    - Express 5.1.0の最新機能を含む公式ドキュメント

21. **[MDN Express チュートリアル：地域図書館ウェブサイト](https://developer.mozilla.org/ja/docs/Learn_web_development/Extensions/Server-side/Express_Nodejs)** 
    - 実践的なフルスタックWebアプリケーション開発

22. **[Microsoft Learn: Node.js チュートリアル](https://learn.microsoft.com/ja-jp/windows/dev-environment/javascript/nodejs-beginners-tutorial)**
    - 初心者向けの体系的な学習パス

#### 動画・コース
23. **[REST API Crash Course](https://www.youtube.com/watch?v=qbLc5a9jdXo)** - YouTube

24. **[Node.js REST API Tutorial](https://www.youtube.com/playlist?list=PL4cUxeGkcC9jBcybHMTIia56aV21o2cZ8)** - YouTube

### 🔧 実践プロジェクト

練習用のプロジェクトアイデア：

1. **ブログAPI**
   - ユーザー管理
   - 記事の CRUD
   - コメント機能

2. **ECサイトAPI**
   - 商品管理
   - 在庫管理
   - 注文処理

3. **タスク管理API**
   - プロジェクト管理
   - タスクの CRUD
   - 進捗管理

---

## 🎯 次のステップ

1. **基本的なCRUD APIを実装** - まずはシンプルなAPIから
2. **認証機能の実装** - JWT認証を組み込む
3. **エラーハンドリングの改善** - 統一されたエラーレスポンス
4. **テストの書き方を学習** - Jest, Supertest等
5. **データベース設計** - 正規化、インデックス設計
6. **デプロイメント** - Docker, CI/CD

---

## 💡 フロントエンドエンジニアへのアドバイス

REST API設計を学ぶことで：

- **フロントエンドの実装が楽になる** - 予測可能なAPIインターフェース
- **バックエンドエンジニアとの連携が改善** - 共通言語での議論
- **フルスタック開発への道** - 両方の視点を理解
- **API設計への提案力** - より良いAPIの提案ができる

最初は完璧を目指さず、小さなAPIから始めて徐々に改善していきましょう！

---

> **Tags**: #REST-API #Backend #Node.js #Express #HTTP #API設計 #フロントエンド #フルスタック #学習ノート #技術 #バックエンド学習