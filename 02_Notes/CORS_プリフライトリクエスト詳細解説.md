# CORS プリフライトリクエスト詳細解説

> **学習日**: 2025-09-20  
> **難易度**: 中級  
> **関連技術**: HTTP、CORS、Web API、ブラウザセキュリティ

## 🎯 プリフライトリクエストとは

**プリフライトリクエスト**（Preflight Request）は、ブラウザが**実際のリクエストを送信する前**に、そのリクエストが許可されているかをサーバーに確認するためのHTTP OPTIONSリクエストです。

### なぜ必要なのか？

```javascript
// 危険な可能性があるリクエスト例
fetch('https://bank-api.com/transfer', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': 'secret123'
  },
  body: JSON.stringify({
    to: 'attacker-account',
    amount: 1000000
  })
});
```

ブラウザは「このリクエストは本当に安全か？」を事前確認します。

## 🔍 プリフライトが発生する条件

### 1. シンプルリクエスト vs プリフライトリクエスト

#### シンプルリクエスト（プリフライト不要）
```javascript
// ✅ プリフライト不要
fetch('/api/users', {
  method: 'GET' // GET, POST, HEAD のみ
});

fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded' // 許可されたヘッダー
  }
});
```

#### プリフライトリクエスト（事前確認必要）
```javascript
// ❌ プリフライト必要
fetch('/api/users', {
  method: 'PUT' // PUT, DELETE, PATCH など
});

fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json' // カスタムヘッダー
  }
});

fetch('/api/users', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer token123' // カスタムヘッダー
  }
});
```

### 2. 詳細な発生条件

#### HTTPメソッド
```javascript
// プリフライト不要
['GET', 'HEAD', 'POST']

// プリフライト必要
['PUT', 'DELETE', 'PATCH', 'OPTIONS', 'CONNECT', 'TRACE']
```

#### Content-Type
```javascript
// プリフライト不要
'application/x-www-form-urlencoded'
'multipart/form-data'
'text/plain'

// プリフライト必要
'application/json'        // ← 最も一般的
'application/xml'
'text/xml'
```

#### リクエストヘッダー
```javascript
// プリフライト不要（シンプルヘッダー）
'Accept'
'Accept-Language'
'Content-Language'
'Content-Type' // (上記の許可された値のみ)

// プリフライト必要（カスタムヘッダー）
'Authorization'           // ← 認証で頻繁に使用
'X-API-Key'
'X-Requested-With'
'X-Custom-Header'
```

## 📡 プリフライトリクエストの流れ

### 1. 実際の通信例

```javascript
// フロントエンド (http://localhost:3000)
fetch('http://localhost:5000/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer eyJhbGc...'
  },
  body: JSON.stringify({ name: 'John Doe' })
});
```

### 2. ステップ1: プリフライトリクエスト
```http
OPTIONS /api/users HTTP/1.1
Host: localhost:5000
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

### 3. ステップ2: サーバーレスポンス
```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

### 4. ステップ3: 実際のリクエスト
```http
POST /api/users HTTP/1.1
Host: localhost:5000
Origin: http://localhost:3000
Content-Type: application/json
Authorization: Bearer eyJhbGc...

{"name": "John Doe"}
```

### 5. ステップ4: 最終レスポンス
```http
HTTP/1.1 201 Created
Access-Control-Allow-Origin: http://localhost:3000
Content-Type: application/json

{"id": 123, "name": "John Doe"}
```

## 🛠️ サーバー側での実装

### 1. Express + cors パッケージ
```javascript
const cors = require('cors');

// 基本設定
app.use(cors({
  origin: 'http://localhost:3000',
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400 // プリフライトキャッシュ時間
}));
```

### 2. 手動でのプリフライト処理
```javascript
app.options('/api/*', (req, res) => {
  res.header('Access-Control-Allow-Origin', 'http://localhost:3000');
  res.header('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE,OPTIONS');
  res.header('Access-Control-Allow-Headers', 'Content-Type,Authorization');
  res.header('Access-Control-Max-Age', '86400');
  res.sendStatus(200);
});

// 実際のAPI
app.post('/api/users', (req, res) => {
  res.header('Access-Control-Allow-Origin', 'http://localhost:3000');
  // ユーザー作成処理
  res.json({ success: true });
});
```

### 3. 条件付きプリフライト処理
```javascript
app.use((req, res, next) => {
  const origin = req.headers.origin;
  const allowedOrigins = ['http://localhost:3000', 'https://myapp.com'];
  
  if (allowedOrigins.includes(origin)) {
    res.header('Access-Control-Allow-Origin', origin);
  }
  
  if (req.method === 'OPTIONS') {
    res.header('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE');
    res.header('Access-Control-Allow-Headers', 'Content-Type,Authorization');
    res.header('Access-Control-Max-Age', '3600');
    return res.sendStatus(200);
  }
  
  next();
});
```

## ⚡ パフォーマンス最適化

### 1. プリフライトキャッシュ
```javascript
// Max-Ageを設定してキャッシュ効率化
app.use(cors({
  maxAge: 86400 // 24時間キャッシュ
}));

// ブラウザは24時間以内なら同じリクエストでプリフライトを省略
```

### 2. キャッシュの確認方法
```javascript
// Chrome DevTools → Network タブ
// 1回目: OPTIONS リクエストが表示される
// 2回目: OPTIONS リクエストが省略される（キャッシュ効果）

console.log('1回目のリクエスト');
fetch('/api/users', { /* ... */ }); // OPTIONS + POST

console.log('2回目のリクエスト（24時間以内）');
fetch('/api/users', { /* ... */ }); // POST のみ
```

### 3. 不要なプリフライトの回避
```javascript
// ❌ 非効率：毎回プリフライト
fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  }
});

// ✅ 効率的：プリフライト不要
const formData = new FormData();
formData.append('name', 'John');
fetch('/api/users', {
  method: 'POST',
  body: formData // multipart/form-data
});
```

## 🚨 よくある問題とトラブルシューティング

### 1. プリフライトが失敗する場合
```javascript
// エラー例
// "Access to fetch at 'http://localhost:5000/api/users' from origin 
//  'http://localhost:3000' has been blocked by CORS policy: 
//  Response to preflight request doesn't pass access control check"

// 原因と解決法
app.use(cors({
  origin: 'http://localhost:3000', // オリジンを正確に指定
  methods: ['POST'], // メソッドを明示的に許可
  allowedHeaders: ['Content-Type', 'Authorization'] // ヘッダーを許可
}));
```

### 2. 認証ヘッダーの問題
```javascript
// ❌ 失敗例
fetch('/api/protected', {
  headers: {
    'Authorization': 'Bearer token123',
    'X-Custom-Header': 'value' // 許可されていない
  }
});

// ✅ 成功例
app.use(cors({
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Custom-Header']
}));
```

### 3. プリフライトリクエストのデバッグ
```javascript
// サーバー側でのログ出力
app.use((req, res, next) => {
  if (req.method === 'OPTIONS') {
    console.log('=== PREFLIGHT REQUEST ===');
    console.log('Origin:', req.headers.origin);
    console.log('Method:', req.headers['access-control-request-method']);
    console.log('Headers:', req.headers['access-control-request-headers']);
    console.log('========================');
  }
  next();
});
```

## 🔒 セキュリティ考慮事項

### 1. プリフライトの意義
```javascript
// プリフライトリクエストは以下を防ぐ
// 1. CSRF攻撃
// 2. 意図しない副作用のあるリクエスト
// 3. 機密データの漏洩

// 悪意のあるサイトからの攻撃例
// https://evil-site.com のスクリプト
fetch('https://bank-api.com/transfer', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ to: 'attacker', amount: 1000 })
});
// → プリフライトでブロックされる
```

### 2. 適切なプリフライト設定
```javascript
// ❌ 危険：全許可
app.use(cors({
  origin: '*',
  allowedHeaders: '*',
  methods: '*'
}));

// ✅ 安全：必要最小限
app.use(cors({
  origin: ['https://trusted-domain.com'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  methods: ['GET', 'POST']
}));
```

## 📊 プリフライトリクエストの判定フローチャート

```
リクエスト送信
    ↓
シンプルリクエスト？
    ↓           ↓
   YES         NO
    ↓           ↓
直接送信    プリフライト送信
    ↓           ↓
   完了     許可される？
              ↓        ↓
             YES      NO
              ↓        ↓
           実際送信   エラー
              ↓
             完了
```

### 判定条件の詳細
```javascript
function needsPreflight(method, headers, contentType) {
  // 1. メソッドチェック
  if (!['GET', 'HEAD', 'POST'].includes(method)) {
    return true;
  }
  
  // 2. Content-Typeチェック
  const allowedContentTypes = [
    'application/x-www-form-urlencoded',
    'multipart/form-data',
    'text/plain'
  ];
  if (contentType && !allowedContentTypes.includes(contentType)) {
    return true;
  }
  
  // 3. ヘッダーチェック
  const simpleHeaders = [
    'accept', 'accept-language', 'content-language', 'content-type'
  ];
  for (let header of Object.keys(headers)) {
    if (!simpleHeaders.includes(header.toLowerCase())) {
      return true;
    }
  }
  
  return false;
}
```

## 🧪 実験・検証方法

### 1. Chrome DevToolsでの確認
```javascript
// 1. Network タブを開く
// 2. 以下のコードを実行
fetch('http://localhost:5000/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'Test' })
});

// 3. Network タブで以下を確認
// - OPTIONS リクエスト（プリフライト）
// - POST リクエスト（実際のリクエスト）
```

### 2. カールコマンドでの検証
```bash
# プリフライトリクエストの手動送信
curl -X OPTIONS \
  -H "Origin: http://localhost:3000" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type" \
  http://localhost:5000/api/users

# レスポンスヘッダーの確認
# Access-Control-Allow-Origin: http://localhost:3000
# Access-Control-Allow-Methods: GET,POST,PUT,DELETE
# Access-Control-Allow-Headers: Content-Type
```

## 💡 実践的なTips

### 1. 開発時のデバッグ
```javascript
// プリフライト専用のミドルウェア
app.use((req, res, next) => {
  if (req.method === 'OPTIONS') {
    console.log(`🔍 PREFLIGHT: ${req.path}`);
    console.log(`   Origin: ${req.headers.origin}`);
    console.log(`   Method: ${req.headers['access-control-request-method']}`);
    console.log(`   Headers: ${req.headers['access-control-request-headers']}`);
  }
  next();
});
```

### 2. 本番環境での最適化
```javascript
// 環境別設定
const corsOptions = {
  origin: process.env.NODE_ENV === 'production' 
    ? process.env.ALLOWED_ORIGINS?.split(',')
    : ['http://localhost:3000'],
  maxAge: process.env.NODE_ENV === 'production' ? 86400 : 0
};

app.use(cors(corsOptions));
```

### 3. プリフライトキャッシュの戦略
```javascript
// APIの種類に応じてキャッシュ時間を調整
app.use('/api/auth', cors({ maxAge: 3600 }));    // 認証API: 1時間
app.use('/api/data', cors({ maxAge: 86400 }));   // データAPI: 24時間
app.use('/api/admin', cors({ maxAge: 0 }));      // 管理API: キャッシュなし
```

---

> **Tags**: #CORS #プリフライト #HTTP #Webセキュリティ #ブラウザ #API #学習ノート #技術 #バックエンド学習

> **関連ページ**: [[REST_API設計_基礎と実践]] | [[WebセキュリティとHTTPS]] | [[HTTPプロトコル基礎]]