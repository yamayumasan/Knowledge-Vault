# JWT認証システム完全ガイド 🔐

> **学習日**: 2025-09-20  
> **難易度**: 中級〜上級  
> **実装参考**: meal-voteプロジェクト  
> **関連技術**: JWT、Node.js、TypeScript、Express、bcrypt

## 🎯 JWT認証システムとは

**JSON Web Token (JWT)** は、当事者間で安全に情報を送信するためのオープンスタンダード（RFC 7519）です。特にWebアプリケーションの認証・認可で広く使用されています。

### なぜJWTが使われるのか？

```javascript
// 従来のセッション認証の問題
// ❌ サーバーがセッション状態を保持する必要
// ❌ スケーラビリティの問題
// ❌ モバイルアプリとの相性が悪い

// ✅ JWTの利点
// ✅ ステートレス（サーバーが状態を保持しない）
// ✅ 分散システムに適している
// ✅ モバイル・SPA・マイクロサービスに最適
```

## 🔍 JWTの構造

### 基本構造
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjMiLCJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20iLCJpYXQiOjE2MzQwMDAwMDAsImV4cCI6MTYzNDAwMzYwMH0.signature
    ┌─────────── HEADER ──────────┐ ┌────────────── PAYLOAD ─────────────┐ ┌─ SIGNATURE ─┐
```

### 1. Header（ヘッダー）
```json
{
  "alg": "HS256",     // 署名アルゴリズム
  "typ": "JWT"        // トークンタイプ
}
```

### 2. Payload（ペイロード）
```json
{
  "userId": "123",
  "email": "user@example.com",
  "role": "admin",
  "familyId": "family_456",
  "iat": 1634000000,    // 発行時刻
  "exp": 1634003600,    // 有効期限
  "iss": "meal-vote-api", // 発行者
  "aud": "meal-vote-app"  // 対象者
}
```

### 3. Signature（署名）
```javascript
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

## 🏗️ meal-voteプロジェクトの実装分析

### アーキテクチャ概要
```
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Client    │───▶│  AuthController │───▶│   AuthService   │
│ (Frontend)  │    │                 │    │                 │
└─────────────┘    └─────────────────┘    └─────────────────┘
                            │                        │
                            ▼                        ▼
                   ┌─────────────────┐    ┌─────────────────┐
                   │ Auth Middleware │    │   JWT Utils     │
                   │                 │    │                 │
                   └─────────────────┘    └─────────────────┘
```

### 1. JWT設定・生成（jwt.ts）

#### トークンペア生成
```typescript
export interface TokenPair {
  accessToken: string;  // 短期間（15分〜1時間）
  refreshToken: string; // 長期間（7日〜30日）
}

export const generateTokens = (payload: JWTPayload): TokenPair => {
  const accessToken = jwt.sign(payload, config.jwtSecret, {
    expiresIn: config.jwtExpire,        // '15m'
    issuer: 'meal-vote-api',            // 発行者識別
    audience: 'meal-vote-app'           // 対象アプリケーション
  });

  const refreshToken = jwt.sign(
    { userId: payload.userId },          // 最小限の情報のみ
    config.jwtRefreshSecret,            // 異なるシークレット
    {
      expiresIn: config.jwtRefreshExpire, // '7d'
      issuer: 'meal-vote-api',
      audience: 'meal-vote-app'
    }
  );

  return { accessToken, refreshToken };
};
```

#### トークン検証
```typescript
export const verifyAccessToken = (token: string): JWTPayload => {
  try {
    const decoded = jwt.verify(token, config.jwtSecret, {
      issuer: 'meal-vote-api',
      audience: 'meal-vote-app'
    }) as JWTPayload;
    
    return decoded;
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new Error('Access token expired');
    } else if (error instanceof jwt.JsonWebTokenError) {
      throw new Error('Invalid access token');
    } else {
      throw new Error('Token verification failed');
    }
  }
};
```

#### Authorization ヘッダー処理
```typescript
export const extractTokenFromHeader = (authHeader: string | undefined): string | null => {
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return null;
  }
  
  return authHeader.substring(7); // "Bearer "を除去
};
```

### 2. 認証サービス（authService.ts）

#### ユーザー登録
```typescript
export class AuthService {
  static async register(data: RegisterData): Promise<AuthResult> {
    const { email, password, name, familyName } = data;

    // 重複チェック
    const existingUser = await User.findOne({ email: email.toLowerCase() });
    if (existingUser) {
      throw new Error('User with this email already exists');
    }

    // 家族作成
    const family = new Family({
      name: familyName,
      adminId: null,  // 後で設定
      memberIds: []
    });
    await family.save();

    // ユーザー作成（パスワードは自動でハッシュ化される）
    const user = new User({
      email: email.toLowerCase(),
      password,                     // Mongooseのpre-saveでハッシュ化
      name,
      role: 'admin',               // 最初のユーザーは管理者
      familyId: family._id,
      preferences: {},
      settings: {
        mealPlanFrequency: 'weekly',
        notificationEnabled: true,
        notificationTimes: {
          mealSuggestion: '20:00',
          votingReminder: '18:00',
          mealConfirmation: '07:00'
        }
      }
    });

    await user.save();

    // 家族情報更新
    family.adminId = user._id;
    family.memberIds.push(user._id);
    await family.save();

    // JWTペイロード作成
    const payload: JWTPayload = {
      userId: user._id.toString(),
      email: user.email,
      role: user.role,
      familyId: user.familyId.toString()
    };

    const tokens = generateTokens(payload);

    // パスワードを除いてユーザー情報を返す
    const userResponse = await User.findById(user._id)
      .populate('familyId', 'name adminId memberIds');

    return {
      user: userResponse,
      tokens
    };
  }
}
```

#### ログイン処理
```typescript
static async login(data: LoginData): Promise<AuthResult> {
  const { email, password } = data;

  // パスワード付きでユーザー検索
  const user = await (User as any).findByEmailWithPassword(email.toLowerCase());
  if (!user) {
    throw new Error('Invalid email or password');
  }

  // パスワード比較
  const isValidPassword = await user.comparePassword(password);
  if (!isValidPassword) {
    throw new Error('Invalid email or password');
  }

  // トークン生成
  const payload: JWTPayload = {
    userId: user._id.toString(),
    email: user.email,
    role: user.role,
    familyId: user.familyId.toString()
  };

  const tokens = generateTokens(payload);

  const userResponse = await User.findById(user._id)
    .populate('familyId', 'name adminId memberIds');

  return {
    user: userResponse,
    tokens
  };
}
```

#### リフレッシュトークン処理
```typescript
static async refreshTokens(refreshToken: string): Promise<TokenPair> {
  try {
    // リフレッシュトークン検証
    const decoded = verifyRefreshToken(refreshToken);

    // ユーザー存在確認
    const user = await User.findById(decoded.userId);
    if (!user) {
      throw new Error('User not found');
    }

    // 新しいトークンペア生成
    const payload: JWTPayload = {
      userId: user._id.toString(),
      email: user.email,
      role: user.role,
      familyId: user.familyId.toString()
    };

    return generateTokens(payload);
  } catch (error: any) {
    throw new Error('Invalid refresh token');
  }
}
```

### 3. 認証ミドルウェア（auth.ts）

#### 基本認証ミドルウェア
```typescript
export const authenticate = async (
  req: AuthenticatedRequest, 
  res: Response, 
  next: NextFunction
): Promise<void> => {
  try {
    // Authorization ヘッダーからトークン抽出
    const token = extractTokenFromHeader(req.headers.authorization);
    
    if (!token) {
      res.status(401).json({
        success: false,
        error: 'Access token required'
      });
      return;
    }

    // トークン検証
    const decoded = verifyAccessToken(token);
    
    // ユーザー情報取得・付加
    const user = await User.findById(decoded.userId)
      .populate('familyId', 'name adminId memberIds');
    
    if (!user) {
      res.status(401).json({
        success: false,
        error: 'User not found'
      });
      return;
    }

    req.user = user;  // リクエストオブジェクトにユーザー情報を追加
    next();
  } catch (error: any) {
    res.status(401).json({
      success: false,
      error: error.message || 'Authentication failed'
    });
  }
};
```

#### 役割ベースアクセス制御
```typescript
export const requireAdmin = async (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
): Promise<void> => {
  try {
    if (!req.user) {
      res.status(401).json({
        success: false,
        error: 'Authentication required'
      });
      return;
    }

    if (req.user.role !== 'admin') {
      res.status(403).json({
        success: false,
        error: 'Admin access required'
      });
      return;
    }

    next();
  } catch (error: any) {
    res.status(500).json({
      success: false,
      error: 'Authorization check failed'
    });
  }
};
```

#### 家族メンバー認証
```typescript
export const requireFamilyMember = async (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
): Promise<void> => {
  try {
    if (!req.user) {
      res.status(401).json({
        success: false,
        error: 'Authentication required'
      });
      return;
    }

    const familyId = req.params.familyId || req.body.familyId || req.query.familyId;
    
    if (!familyId) {
      res.status(400).json({
        success: false,
        error: 'Family ID required'
      });
      return;
    }

    if (req.user.familyId.toString() !== familyId) {
      res.status(403).json({
        success: false,
        error: 'Access denied: not a family member'
      });
      return;
    }

    next();
  } catch (error: any) {
    res.status(500).json({
      success: false,
      error: 'Family member check failed'
    });
  }
};
```

#### オプション認証
```typescript
export const optionalAuth = async (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
): Promise<void> => {
  try {
    const token = extractTokenFromHeader(req.headers.authorization);
    
    if (token) {
      try {
        const decoded = verifyAccessToken(token);
        const user = await User.findById(decoded.userId)
          .populate('familyId', 'name adminId memberIds');
        
        if (user) {
          req.user = user;
        }
      } catch (error) {
        // オプション認証では無視
      }
    }

    next();
  } catch (error: any) {
    next();
  }
};
```

### 4. パスワードハッシュ化（User.ts）

#### Mongooseスキーマでの自動ハッシュ化
```typescript
// Pre-save ミドルウェアでパスワードハッシュ化
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  try {
    const salt = await bcrypt.genSalt(12);      // ソルトラウンド12
    this.password = await bcrypt.hash(this.password, salt);
    next();
  } catch (error) {
    next(error as any);
  }
});

// パスワード比較インスタンスメソッド
userSchema.methods.comparePassword = async function(candidatePassword: string): Promise<boolean> {
  try {
    return await bcrypt.compare(candidatePassword, this.password);
  } catch (error) {
    throw error;
  }
};

// パスワード付きでのユーザー検索（静的メソッド）
userSchema.statics.findByEmailWithPassword = function(email: string) {
  return this.findOne({ email }).select('+password');  // 通常は除外されるpasswordを含める
};
```

## 🔒 セキュリティベストプラクティス

### 1. シークレット管理
```typescript
// ❌ 危険
const JWT_SECRET = 'simple-password';

// ✅ 安全
const JWT_SECRET = process.env.JWT_SECRET || crypto.randomBytes(64).toString('hex');
const JWT_REFRESH_SECRET = process.env.JWT_REFRESH_SECRET || crypto.randomBytes(64).toString('hex');

// 環境変数例
// JWT_SECRET=a9b8c7d6e5f4g3h2i1j0k9l8m7n6o5p4q3r2s1t0u9v8w7x6y5z4a3b2c1d0
// JWT_REFRESH_SECRET=z9y8x7w6v5u4t3s2r1q0p9o8n7m6l5k4j3i2h1g0f9e8d7c6b5a4z3y2x1w0
```

### 2. トークン有効期限
```typescript
// 推奨設定
const config = {
  jwtExpire: '15m',          // アクセストークン: 15分
  jwtRefreshExpire: '7d',    // リフレッシュトークン: 7日
};

// 高セキュリティが必要な場合
const config = {
  jwtExpire: '5m',           // 5分
  jwtRefreshExpire: '1d',    // 1日
};
```

### 3. HTTPS必須
```typescript
// 本番環境では必須
app.use((req, res, next) => {
  if (process.env.NODE_ENV === 'production' && !req.secure && req.get('x-forwarded-proto') !== 'https') {
    return res.redirect(301, `https://${req.get('host')}${req.url}`);
  }
  next();
});
```

### 4. レート制限
```typescript
import rateLimit from 'express-rate-limit';

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15分
  max: 5,                    // 最大5回の試行
  message: {
    error: 'Too many authentication attempts, please try again later.',
  },
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/api/auth/login', authLimiter);
app.use('/api/auth/register', authLimiter);
```

## 🌊 リクエストフロー詳細

### 1. 登録フロー
```
Client                 Controller              Service                Database
  │                      │                      │                      │
  ├─POST /api/auth/register                     │                      │
  │                      │                      │                      │
  │                      ├─validateInput()      │                      │
  │                      │                      │                      │
  │                      │                ├─checkExistingUser()        │
  │                      │                      │                ├─findOne()
  │                      │                      │                │
  │                      │                      │          ◀──── │
  │                      │                      │                      │
  │                      │                ├─createUser()               │
  │                      │                      │                ├─save()
  │                      │                      │                │
  │                      │                      │          ◀──── │
  │                      │                      │                      │
  │                      │                ├─generateTokens()           │
  │                      │                      │                      │
  │                ◀──── │          ◀──── │                      │
  │                      │                      │                      │
  ◀──────200 + user + tokens               │                      │
```

### 2. ログインフロー
```
Client                 Controller              Service                Database
  │                      │                      │                      │
  ├─POST /api/auth/login                        │                      │
  │                      │                      │                      │
  │                      ├─validateInput()      │                      │
  │                      │                      │                      │
  │                      │                ├─findUserWithPassword()    │
  │                      │                      │                ├─findOne()
  │                      │                      │                │
  │                      │                      │          ◀──── │
  │                      │                      │                      │
  │                      │                ├─comparePassword()          │
  │                      │                      │                      │
  │                      │                ├─generateTokens()           │
  │                      │                      │                      │
  │                ◀──── │          ◀──── │                      │
  │                      │                      │                      │
  ◀──────200 + user + tokens               │                      │
```

### 3. 認証付きAPIアクセス
```
Client                 Middleware             Controller             Service
  │                      │                      │                      │
  ├─GET /api/protected (Bearer token)           │                      │
  │                      │                      │                      │
  │                ├─extractToken()             │                      │
  │                      │                      │                      │
  │                ├─verifyAccessToken()        │                      │
  │                      │                      │                      │
  │                ├─findUser()                 │                      │
  │                      │                      │                      │
  │                ├─req.user = user            │                      │
  │                      │                      │                      │
  │                      ├─next()───────────────►                     │
  │                      │                      │                      │
  │                      │                      ├─processRequest()     │
  │                      │                      │                ├─logic()
  │                      │                      │                │
  │                      │                      │          ◀──── │
  │                      │                ◀──── │                      │
  │                      │                      │                      │
  ◀──────────────────200 + data            │                      │
```

## ⚡ パフォーマンス最適化

### 1. トークンキャッシュ
```typescript
import NodeCache from 'node-cache';

const tokenCache = new NodeCache({ 
  stdTTL: 600, // 10分
  checkperiod: 120 
});

export const verifyAccessTokenCached = (token: string): JWTPayload => {
  // キャッシュチェック
  const cached = tokenCache.get(token);
  if (cached) {
    return cached as JWTPayload;
  }

  // 検証実行
  const decoded = verifyAccessToken(token);
  
  // キャッシュに保存
  tokenCache.set(token, decoded);
  
  return decoded;
};
```

### 2. ユーザー情報キャッシュ
```typescript
const userCache = new NodeCache({ 
  stdTTL: 300, // 5分
  checkperiod: 60 
});

export const getUserCached = async (userId: string) => {
  const cached = userCache.get(userId);
  if (cached) {
    return cached;
  }

  const user = await User.findById(userId).populate('familyId');
  userCache.set(userId, user);
  
  return user;
};
```

### 3. データベースインデックス
```typescript
// ユーザースキーマにインデックス追加
userSchema.index({ email: 1 });        // 一意制約も兼ねる
userSchema.index({ familyId: 1 });     // 家族による検索最適化
userSchema.index({ role: 1 });         // 役割による検索最適化
userSchema.index({ 
  email: 1, 
  familyId: 1 
});                                    // 複合インデックス
```

## 🚨 よくある問題とトラブルシューティング

### 1. "Token expired" エラー
```typescript
// 問題: アクセストークンの有効期限が短すぎる
// 解決: リフレッシュトークンで自動更新

const useAuthToken = () => {
  const [token, setToken] = useState(localStorage.getItem('accessToken'));
  const [refreshToken, setRefreshToken] = useState(localStorage.getItem('refreshToken'));

  const refreshAccessToken = async () => {
    try {
      const response = await fetch('/api/auth/refresh', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken })
      });

      const data = await response.json();
      
      if (data.success) {
        setToken(data.data.tokens.accessToken);
        setRefreshToken(data.data.tokens.refreshToken);
        localStorage.setItem('accessToken', data.data.tokens.accessToken);
        localStorage.setItem('refreshToken', data.data.tokens.refreshToken);
        return data.data.tokens.accessToken;
      }
    } catch (error) {
      // ログアウト処理
      logout();
    }
  };

  return { token, refreshAccessToken };
};
```

### 2. "Invalid signature" エラー
```typescript
// 問題: 異なるシークレットでトークンが生成・検証されている
// 確認ポイント:
console.log('JWT_SECRET:', process.env.JWT_SECRET?.substring(0, 10) + '...');
console.log('NODE_ENV:', process.env.NODE_ENV);

// 本番と開発で同じシークレットを使用しているか確認
```

### 3. CORS エラー with Authorization ヘッダー
```typescript
// 問題: Authorization ヘッダーがプリフライトでブロックされる
app.use(cors({
  origin: ['http://localhost:3000', 'https://yourdomain.com'],
  allowedHeaders: ['Content-Type', 'Authorization'], // ← 重要
  credentials: true
}));
```

### 4. メモリリーク（キャッシュ）
```typescript
// 問題: トークンキャッシュがメモリリークを起こす
const tokenCache = new NodeCache({ 
  stdTTL: 600,
  checkperiod: 120,
  useClones: false,    // メモリ使用量削減
  maxKeys: 1000        // 最大キー数制限
});

// 定期的なキャッシュクリア
setInterval(() => {
  tokenCache.flushAll();
}, 3600000); // 1時間ごと
```

## 📊 セキュリティチェックリスト

### ✅ 必須項目
- [ ] HTTPS通信（本番環境）
- [ ] 強力なシークレット（256ビット以上）
- [ ] 適切なトークン有効期限設定
- [ ] パスワードの適切なハッシュ化（bcrypt、saltラウンド10以上）
- [ ] レート制限実装
- [ ] 入力バリデーション
- [ ] SQLインジェクション対策（Mongoose使用時は自動）

### ✅ 推奨項目
- [ ] リフレッシュトークンローテーション
- [ ] トークンブラックリスト機能
- [ ] ログイン試行回数制限
- [ ] 多要素認証（2FA）準備
- [ ] セキュリティヘッダー（helmet）
- [ ] APIバージョニング
- [ ] エラーメッセージの情報漏洩防止

## 🔗 フロントエンド連携例

### React + Axios インターセプター
```typescript
import axios from 'axios';

const api = axios.create({
  baseURL: '/api'
});

// リクエストインターセプター（トークン自動付与）
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// レスポンスインターセプター（トークン自動更新）
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const original = error.config;

    if (error.response?.status === 401 && !original._retry) {
      original._retry = true;

      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const response = await axios.post('/api/auth/refresh', {
          refreshToken
        });

        const { accessToken, refreshToken: newRefreshToken } = response.data.data.tokens;
        
        localStorage.setItem('accessToken', accessToken);
        localStorage.setItem('refreshToken', newRefreshToken);

        original.headers.Authorization = `Bearer ${accessToken}`;
        return api(original);
      } catch (refreshError) {
        // リフレッシュ失敗→ログアウト
        localStorage.removeItem('accessToken');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';
      }
    }

    return Promise.reject(error);
  }
);
```

### React Context での認証状態管理
```typescript
interface AuthContextType {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  loading: boolean;
}

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      api.get('/auth/me')
        .then(response => setUser(response.data.data.user))
        .catch(() => {
          localStorage.removeItem('accessToken');
          localStorage.removeItem('refreshToken');
        })
        .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);

  const login = async (email: string, password: string) => {
    const response = await api.post('/auth/login', { email, password });
    const { user, tokens } = response.data.data;
    
    localStorage.setItem('accessToken', tokens.accessToken);
    localStorage.setItem('refreshToken', tokens.refreshToken);
    setUser(user);
  };

  const logout = () => {
    localStorage.removeItem('accessToken');
    localStorage.removeItem('refreshToken');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

## 🎯 次のステップ

1. **実装練習**
   - 簡単なログイン・登録API作成
   - 認証ミドルウェア実装
   - フロントエンドとの連携

2. **セキュリティ強化**
   - OAuth 2.0 / OpenID Connect学習
   - 多要素認証（2FA）実装
   - セキュリティヘッダー強化

3. **スケーラビリティ**
   - Redis での セッション・キャッシュ管理
   - マイクロサービス間認証
   - JWT vs Session 適切な選択

4. **監視・ログ**
   - 認証ログの収集・分析
   - 異常アクセス検知
   - パフォーマンス監視

---

> **Tags**: #JWT #認証 #セキュリティ #Node.js #TypeScript #Express #bcrypt #認可 #バックエンド学習 #meal-vote #実装解析

> **関連ページ**: [[REST_API設計_基礎と実践]] | [[CORS_プリフライトリクエスト詳細解説]] | [[Mongooseデータベース設計]]