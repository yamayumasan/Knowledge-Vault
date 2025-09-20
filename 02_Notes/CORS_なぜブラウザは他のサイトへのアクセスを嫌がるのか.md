# CORSとプリフライトリクエスト〜「Access-Control-Allow-Origin」エラーで3日悩んだ話〜

> **対象者**: 「なぜlocalhostだとエラーになるの？」と困っている方  
> **私の経験**: フロントエンドとバックエンドを別々に開発して、初めてCORSの壁にぶつかった  
> **作成日**: 2025-09-20

## 📖 あの日、突然エラーが出た

React（localhost:3000）とExpress（localhost:5000）で開発していた、ある金曜日の夕方。

```javascript
// Reactアプリから
fetch('http://localhost:5000/api/users')
  .then(res => res.json())
  .then(data => console.log(data));
```

**ブラウザ：**
```
Access to fetch at 'http://localhost:5000/api/users' from origin 
'http://localhost:3000' has been blocked by CORS policy: 
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

私：「は？localhostなのに、なんでブロックされるの？？？」

これが、私とCORSとの長い戦いの始まりでした...

---

## そもそも、なぜブラウザはこんなに厳しいのか

### インターネットの危険な世界

想像してください。あなたが銀行のサイト（bank.com）にログインしている状態で、
別のタブで怪しいサイト（evil.com）を開いてしまった場合...

```javascript
// もしCORSがなかったら、evil.comはこんなことができてしまう

// evil.comのJavaScript
fetch('https://bank.com/api/transfer', {
  method: 'POST',
  credentials: 'include',  // あなたのCookieを使って
  body: JSON.stringify({
    to: 'hacker-account',
    amount: 1000000
  })
});

// 😱 あなたの銀行口座から100万円が盗まれる！
```

**これを防ぐのが「同一オリジンポリシー」です。**

### オリジンとは何か？

```javascript
// オリジン = プロトコル + ドメイン + ポート

https://example.com:443
  ↑        ↑         ↑
プロトコル ドメイン  ポート

// 同じオリジン？
https://example.com      → https://example.com      ✅ 同じ
https://example.com      → http://example.com       ❌ プロトコルが違う
https://example.com      → https://api.example.com  ❌ サブドメインも違う扱い
http://localhost:3000    → http://localhost:5000    ❌ ポートが違う
```

**なぜlocalhost:3000とlocalhost:5000が違うオリジンなの？**

ブラウザ：「ポートが違う = 別のアプリケーション = 信用できない」

私：「でも両方とも自分のPCじゃん...」

ブラウザ：「ルールはルール。例外は認めない。」

---

## CORSが解決する問題と、新たに生む問題

### CORS（Cross-Origin Resource Sharing）の仕組み

CORSは「信頼できる相手からのアクセスは許可する」仕組みです。

```javascript
// サーバー（localhost:5000）が言う
「localhost:3000からのアクセスは許可するよ」

// ブラウザ
「了解！じゃあlocalhost:3000からのfetchは通すね」
```

### でも、単純じゃない「プリフライトリクエスト」

ある日、私はJSONをPOSTしようとしました。

```javascript
fetch('http://localhost:5000/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ name: 'Taro' })
});
```

ChromeのNetworkタブを見ると...

**なんと2回リクエストが飛んでいる！？**

1. OPTIONS /api/users （これは何？）
2. POST /api/users （本来のリクエスト）

---

## プリフライトリクエスト：ブラウザの「事前確認」

### なぜブラウザは事前確認をするのか

現実世界で例えると：

```
あなた：「すみません、写真撮ってもいいですか？」（プリフライト）
相手：「いいですよ」
あなた：「じゃあ撮りますね」（実際のリクエスト）

vs

あなた：いきなり写真を撮る（プリフライトなし）
相手：「ちょっと！勝手に撮らないで！」
```

### プリフライトが必要な条件

私は実験して、ようやく理解しました：

**プリフライト不要（シンプルリクエスト）：**
```javascript
// 1. GETリクエスト
fetch('http://localhost:5000/api/users');

// 2. POSTでもContent-Typeが特定の値
fetch('http://localhost:5000/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: 'name=Taro&age=25'
});
```

**プリフライト必要：**
```javascript
// 1. JSONを送る（現代のAPIでは普通）
fetch('http://localhost:5000/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'  // ← これ！
  },
  body: JSON.stringify({ name: 'Taro' })
});

// 2. カスタムヘッダーを使う
fetch('http://localhost:5000/api/users', {
  headers: {
    'Authorization': 'Bearer token123'  // ← これも！
  }
});

// 3. PUT, DELETE, PATCHメソッド
fetch('http://localhost:5000/api/users/123', {
  method: 'DELETE'  // ← これも！
});
```

### なぜJSONを送るとプリフライトが必要なの？

**歴史的な理由：**

2000年代初頭、Webは主にHTMLフォームでデータを送っていました。

```html
<!-- 昔のWeb -->
<form action="/submit" method="POST">
  <input name="name" value="Taro">
  <button type="submit">送信</button>
</form>
<!-- Content-Type: application/x-www-form-urlencoded -->
```

その頃のブラウザ：「フォームのPOSTは安全だと分かってる」

2010年代、JSONが主流になると...

```javascript
// 現代のWeb
fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'Taro' })
});
```

ブラウザ：「JSON？新しいやつだ。危険かも。事前確認しよう」

---

## 私が3日悩んだCORSエラーと、その解決方法

### Day 1: 「とりあえずCORSパッケージ入れればいいんでしょ」

```javascript
// Express サーバー
const cors = require('cors');
app.use(cors());

// これで解決！と思ったら...
```

本番環境でセキュリティチームから怒られました。
「全オリジン許可って、セキュリティ的にアウトです」

### Day 2: 「じゃあオリジン指定すればいいのか」

```javascript
app.use(cors({
  origin: 'http://localhost:3000'
}));

// 開発環境はOK！でも本番は...
// origin: 'https://myapp.com'  // 本番用
// あれ、環境によって変えないといけない...？
```

### Day 3: 「認証トークンが送れない！」

```javascript
// フロントエンド
fetch('http://localhost:5000/api/protected', {
  headers: {
    'Authorization': 'Bearer token123'
  }
});

// エラー：Request header field authorization is not allowed
```

調べて分かったこと：カスタムヘッダーも許可が必要！

```javascript
app.use(cors({
  origin: 'http://localhost:3000',
  allowedHeaders: ['Content-Type', 'Authorization']  // ← これ！
}));
```

### 最終的な解決策

```javascript
// 環境に応じた設定
const corsOptions = {
  origin: function (origin, callback) {
    const allowedOrigins = [
      'http://localhost:3000',
      'http://localhost:3001',
      'https://myapp.com',
      'https://www.myapp.com'
    ];
    
    // origin が undefined の場合は同一オリジン（Postmanなど）
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('CORS policy violation'));
    }
  },
  credentials: true,  // Cookie送信を許可
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400  // プリフライトの結果を24時間キャッシュ
};

app.use(cors(corsOptions));
```

---

## プリフライトリクエストのパフォーマンス問題

### 問題：APIを呼ぶたびに2回通信している

```javascript
// 毎回プリフライトが発生
await fetch('/api/users', { method: 'POST', ... });     // OPTIONS + POST
await fetch('/api/products', { method: 'POST', ... });  // OPTIONS + POST
await fetch('/api/orders', { method: 'POST', ... });    // OPTIONS + POST

// 3回のAPIコール = 6回の通信！？
```

### 解決策1：プリフライトのキャッシュ

```javascript
app.use(cors({
  maxAge: 86400  // 24時間キャッシュ
}));

// 結果：
// 1回目：OPTIONS + POST
// 2回目：POST のみ（OPTIONSはキャッシュから）
// 3回目：POST のみ（OPTIONSはキャッシュから）
```

### 解決策2：シンプルリクエストで済む場合は使う

```javascript
// プリフライトが必要
fetch('/api/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ query: 'test' })
});

// プリフライト不要にできる場合
fetch('/api/search?query=test', {
  method: 'GET'
});
```

---

## 私が学んだCORSの教訓

### 1. CORSは「ブラウザ」のルール

```bash
# curlやPostmanは普通に動く
curl http://localhost:5000/api/users

# でもブラウザはCORSチェックをする
# これが初心者を混乱させる原因
```

### 2. localhostでも油断できない

開発環境でも本番環境と同じCORS設定をテストすることが大事。

### 3. エラーメッセージをちゃんと読む

```
"No 'Access-Control-Allow-Origin' header"
→ サーバーがCORSヘッダーを返していない

"Request header field X is not allowed"
→ allowedHeaders に X を追加する必要がある

"CORS policy: Request blocked"
→ origin が許可リストにない
```

### 4. セキュリティとのバランス

```javascript
// ❌ 危険：すべて許可
app.use(cors({ origin: '*' }));

// ⚠️ 開発のみ：localhost許可
app.use(cors({ origin: 'http://localhost:3000' }));

// ✅ 本番：明示的に許可
app.use(cors({
  origin: ['https://myapp.com'],
  credentials: true
}));
```

---

## まとめ：CORSは敵じゃない、守護者だ

最初はCORSを恨みました。
「なんでこんな面倒なものが...」

でも今は理解しています。
CORSは私たちのWebアプリケーションを、悪意のあるサイトから守ってくれる守護者なんだと。

プリフライトリクエストも、最初は無駄に見えました。
でも、「危険な操作をする前に、ちゃんと確認する」という、当たり前のことをやっているだけなんです。

次にCORSエラーに出会ったら、イライラする前に思い出してください。
ブラウザはあなたのユーザーを守ろうとしているだけだということを。

---

> **Tags**: #CORS #プリフライト #WebSecurity #API #フロントエンド #バックエンド #学習ノート #エラー解決

> **関連ページ**: [[REST_API設計_基礎と実践]] | [[JWT認証システム完全ガイド]]