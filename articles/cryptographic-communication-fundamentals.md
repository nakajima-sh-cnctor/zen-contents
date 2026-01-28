---
title: "暗号通信の基礎：インターネットの安全を守る仕組みを理解しよう"
emoji: "🔐"
type: "tech"
topics: ["暗号化", "セキュリティ", "HTTPS", "SSL/TLS", "初心者向け"]
published: false
---

# はじめに

「クレジットカード番号を入力して大丈夫かな…」
「このサイト、本当に安全なの？」

オンラインショッピングやネットバンキングを使うとき、こんな不安を感じたことはありませんか？

実は、私たちが毎日安心してインターネットを使えているのは、**暗号通信**という技術のおかげなんです。パスワード、クレジットカード情報、個人的なメッセージ…これらすべてが、暗号化されて守られています。

この記事では、インターネットの安全を支える暗号通信の仕組みを、初心者の方にもわかりやすく解説していきます。難しい数学の話は最小限に、「なぜ必要なのか」「どう動いているのか」を中心にお伝えします。

# 1. 暗号通信とは？

暗号通信とは、**データを第三者に読めない形に変換して送受信する技術**です。

## 手紙で例えると

### 普通の手紙（暗号化なし）

```
差出人: 太郎
宛先: 花子

花子さんへ
来週の金曜日、19時に駅前のカフェで会いましょう。
パスワードは「sakura2024」です。

太郎より
```

この手紙を郵便配達員や誰かが開封したら、内容が全部バレてしまいます。

### 暗号化した手紙

```
差出人: 太郎
宛先: 花子

8fj3k9dj3kf9j3kd9fj3kd9fj3kd9fj3kd9fj3kd9fj3kd9fj3kd
9fj3kd9fj3kd9fj3kd9fj3kd9fj3kd9fj3kd9fj3kd9fj3kd9fj
```

途中で誰かが見ても、意味不明な文字列にしか見えません。でも、花子さんは「鍵」を持っているので、元の文章に戻せます。

インターネット上でも、同じことが行われています。

## なぜ必要なのか？

インターネットでデータを送るとき、実は多くの中継地点を経由します：

```
あなたのPC → ルーター → プロバイダ → 複数のサーバー → 目的地
```

この途中のどこかで、悪意のある人がデータを盗み見ようとするかもしれません。
暗号化していないと、以下のような危険があります：

- **盗聴**: パスワードやクレジットカード情報が盗まれる
- **改ざん**: データを書き換えられる（振込先を変更されるなど）
- **なりすまし**: 偽のサイトに誘導される

暗号通信は、これらの脅威からあなたを守ってくれるのです。

# 2. 暗号化の基本概念

暗号化には、大きく分けて2つの方式があります。

## 2-1. 共通鍵暗号方式（対称暗号）

**同じ鍵で暗号化と復号化を行う方式**です。

### 仕組み

```
平文（元のデータ）
    ↓
[共通鍵で暗号化]
    ↓
暗号文（読めない形）
    ↓
[同じ共通鍵で復号化]
    ↓
平文（元に戻る）
```

### 例え話：南京錠

南京錠を想像してください。

1. 太郎が宝箱に南京錠をかけて、花子に送る
2. 花子は同じ鍵を持っているので、開けられる

**メリット**:
- 処理が速い
- 大量のデータを暗号化するのに適している

**デメリット**:
- 鍵をどうやって安全に相手に渡すか？という問題がある
- 相手が100人いたら、100個の鍵を管理しなければならない

### 代表的なアルゴリズム

- **AES（Advanced Encryption Standard）**: 現在の標準。銀行やクラウドサービスで使われている
- **DES（Data Encryption Standard）**: 古い方式。今は安全性が低いため非推奨
- **ChaCha20**: 高速で、モバイル端末に適している

### JavaScriptでの例（Web Crypto API）

```javascript
// AESで暗号化する例
async function encryptData(text, password) {
  // パスワードから鍵を生成
  const encoder = new TextEncoder();
  const data = encoder.encode(text);
  
  const keyMaterial = await crypto.subtle.importKey(
    'raw',
    encoder.encode(password),
    'PBKDF2',
    false,
    ['deriveBits', 'deriveKey']
  );
  
  const key = await crypto.subtle.deriveKey(
    {
      name: 'PBKDF2',
      salt: encoder.encode('salt'),
      iterations: 100000,
      hash: 'SHA-256'
    },
    keyMaterial,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
  );
  
  // 暗号化
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    data
  );
  
  return { encrypted, iv };
}

// 使用例
const result = await encryptData('秘密のメッセージ', 'my-password-123');
console.log('暗号化されたデータ:', result.encrypted);
```

## 2-2. 公開鍵暗号方式（非対称暗号）

**異なる2つの鍵（公開鍵と秘密鍵）を使う方式**です。

### 仕組み

```
公開鍵（誰でも知っていい）で暗号化
    ↓
暗号文
    ↓
秘密鍵（自分だけが持つ）で復号化
```

### 例え話：郵便ポスト

郵便ポストを想像してください。

1. **公開鍵 = ポストの投函口**: 誰でも手紙を入れられる
2. **秘密鍵 = ポストの鍵**: 郵便局員（持ち主）だけが開けられる

誰でもあなた宛に暗号化したメッセージを送れますが、開けられるのはあなただけです。

### メリットとデメリット

**メリット**:
- 鍵の配送問題が解決（公開鍵は公開してOK）
- 相手が100人いても、自分の秘密鍵1つだけ管理すればいい

**デメリット**:
- 処理が遅い（共通鍵暗号の100〜1000倍遅い）
- 大量のデータの暗号化には不向き

### 代表的なアルゴリズム

- **RSA**: 最も有名。銀行、政府機関で使われている
- **ECC（楕円曲線暗号）**: RSAより短い鍵で同等の強度。スマホに適している
- **EdDSA**: 高速で安全。最新のシステムで採用が増えている

### JavaScriptでの例

```javascript
// RSA鍵ペアを生成
async function generateKeyPair() {
  const keyPair = await crypto.subtle.generateKey(
    {
      name: 'RSA-OAEP',
      modulusLength: 2048,
      publicExponent: new Uint8Array([1, 0, 1]),
      hash: 'SHA-256'
    },
    true,
    ['encrypt', 'decrypt']
  );
  
  return keyPair;
}

// 公開鍵で暗号化
async function encryptWithPublicKey(publicKey, text) {
  const encoder = new TextEncoder();
  const data = encoder.encode(text);
  
  const encrypted = await crypto.subtle.encrypt(
    { name: 'RSA-OAEP' },
    publicKey,
    data
  );
  
  return encrypted;
}

// 秘密鍵で復号化
async function decryptWithPrivateKey(privateKey, encrypted) {
  const decrypted = await crypto.subtle.decrypt(
    { name: 'RSA-OAEP' },
    privateKey,
    encrypted
  );
  
  const decoder = new TextDecoder();
  return decoder.decode(decrypted);
}

// 使用例
const keyPair = await generateKeyPair();
const encrypted = await encryptWithPublicKey(keyPair.publicKey, '秘密のメッセージ');
const decrypted = await decryptWithPrivateKey(keyPair.privateKey, encrypted);
console.log('復号化:', decrypted); // '秘密のメッセージ'
```

## 2-3. ハイブリッド暗号方式

実際のWebサイト（HTTPS）では、**両方の良いとこ取り**をしています。

### 仕組み

1. **公開鍵暗号**で、共通鍵を安全に交換
2. その後は**共通鍵暗号**で高速に通信

```
[最初だけ]
クライアント → 公開鍵暗号で共通鍵を送る → サーバー

[以降の通信]
クライアント ← 共通鍵暗号で高速通信 → サーバー
```

これにより、「鍵の配送問題」と「処理速度」の両方を解決しています。

# 3. HTTPS：Webの暗号通信

普段使っているWebサイトのURLを見てください。`https://` で始まっていますか？
この「s」が、**Secure（安全）**を意味します。

## HTTPとHTTPSの違い

### HTTP（暗号化なし）

```
クライアント → [ユーザー名: admin, パスワード: pass123] → サーバー
                         ↑
                   誰でも読める！
```

### HTTPS（暗号化あり）

```
クライアント → [8fj3k9dj3kf9j3kd9fj3kd...] → サーバー
                         ↑
                   読めない！
```

## SSL/TLSとは

HTTPSを実現しているのが、**SSL/TLS**という技術です。

- **SSL（Secure Sockets Layer）**: 古い名称。今は使われていない
- **TLS（Transport Layer Security）**: 現在の標準（TLS 1.2、TLS 1.3）

一般的には「SSL/TLS」や「SSL証明書」と呼ばれますが、実際にはTLSが使われています。

## TLSハンドシェイク：暗号化の準備

ブラウザがHTTPSサイトに接続するとき、以下のような「握手（ハンドシェイク）」が行われます。

### 1. クライアントからの挨拶

```
クライアント: こんにちは！
              対応している暗号方式はこれです: [AES, RSA, ...]
              TLSバージョン: 1.3
```

### 2. サーバーからの応答

```
サーバー: こんにちは！
          使う暗号方式はAESにしましょう
          これが私の証明書です（公開鍵付き）
```

### 3. 証明書の検証

```
クライアント: （証明書を確認）
              この証明書、信頼できる機関（CA）が発行したものだな。OK！
```

### 4. 共通鍵の生成

```
クライアント: 共通鍵を作りました
              サーバーの公開鍵で暗号化して送ります
              [暗号化された共通鍵]

サーバー: （秘密鍵で復号化）
          共通鍵を受け取りました！
```

### 5. 暗号化通信開始

```
クライアント ← [共通鍵で暗号化された通信] → サーバー
```

この一連の流れは、わずか数ミリ秒で完了します。

## SSL証明書の役割

SSL証明書は、**「このサイトが本物であることの証明書」**です。

### 証明書に含まれる情報

- サイトのドメイン名（example.com）
- 運営者の情報（会社名など）
- 公開鍵
- 有効期限
- 発行した認証局（CA）の署名

### 認証局（CA）とは

**Certificate Authority（認証局）**は、「この証明書は本物です」と保証する第三者機関です。

有名なCA：
- Let's Encrypt（無料）
- DigiCert
- GlobalSign
- Sectigo

ブラウザには、信頼できるCAのリストが組み込まれています。
証明書がそのCAによって署名されていれば、ブラウザは「このサイトは安全」と判断します。

### ブラウザでの確認方法

ChromeやSafariでHTTPSサイトを開いたとき、アドレスバーに**鍵マーク🔒**が表示されます。
これをクリックすると、証明書の詳細を見ることができます。

```javascript
// JavaScriptから証明書情報を取得することはできませんが、
// 接続がHTTPSかどうかは確認できます
if (window.location.protocol === 'https:') {
  console.log('このページは暗号化されています');
} else {
  console.log('⚠️ このページは暗号化されていません');
}
```

# 4. 暗号化の強度

「暗号化されているから絶対安全！」というわけではありません。
暗号の強度は、**鍵の長さ**と**アルゴリズム**によって決まります。

## 鍵の長さ

鍵が長いほど、解読が難しくなります。

### 共通鍵暗号（AES）

- **128ビット**: 十分安全（2^128 = 約340兆の340兆倍の組み合わせ）
- **256ビット**: 超安全（軍事レベル）

### 公開鍵暗号（RSA）

- **2048ビット**: 現在の標準
- **4096ビット**: より安全（ただし処理が遅い）
- **1024ビット以下**: 非推奨（解読される可能性あり）

## 総当たり攻撃（ブルートフォース）

すべての鍵の組み合わせを試す攻撃です。

### AES-128の場合

```
組み合わせ数: 2^128 = 340,282,366,920,938,463,463,374,607,431,768,211,456通り

スーパーコンピュータで1秒間に1兆個試しても、
全部試すのに約100億年かかります（宇宙の年齢より長い！）
```

つまり、現実的には解読不可能です。

## 量子コンピュータの脅威

将来、**量子コンピュータ**が実用化されると、現在の暗号（特にRSA）が破られる可能性があります。

そのため、**耐量子暗号（Post-Quantum Cryptography）**の研究が進められています。

# 5. デジタル署名：改ざん防止の仕組み

暗号化とセットで使われるのが、**デジタル署名**です。

## デジタル署名とは

**「このデータは私が作成したもので、改ざんされていません」と証明する仕組み**です。

### 手書きの署名との違い

| 手書き署名 | デジタル署名 |
|-----------|-------------|
| 誰でも真似できる | 真似できない（秘密鍵が必要） |
| 文書が変わっても署名は同じ | 文書が1文字でも変わると署名が無効になる |

## 仕組み

### 1. 署名の作成

```
元のデータ
    ↓
[ハッシュ関数]
    ↓
ハッシュ値（データの指紋）
    ↓
[秘密鍵で暗号化]
    ↓
デジタル署名
```

### 2. 署名の検証

```
受信したデータ → [ハッシュ関数] → ハッシュ値A

受信した署名 → [公開鍵で復号化] → ハッシュ値B

ハッシュ値Aとハッシュ値Bが一致？
  → YES: データは本物で改ざんされていない ✅
  → NO: データが改ざんされている ❌
```

## ハッシュ関数

**データから固定長の「指紋」を作る関数**です。

### 特徴

1. **一方向性**: ハッシュ値から元のデータは復元できない
2. **決定性**: 同じデータからは必ず同じハッシュ値
3. **衝突耐性**: 異なるデータから同じハッシュ値が出る確率は極めて低い
4. **雪崩効果**: 1文字変わるだけでハッシュ値が大きく変わる

### 代表的なハッシュ関数

- **SHA-256**: 現在の標準（256ビット）
- **SHA-3**: 最新の標準
- **MD5**: 古い方式。今は安全性が低いため非推奨

### JavaScriptでの例

```javascript
// SHA-256ハッシュを計算
async function calculateHash(text) {
  const encoder = new TextEncoder();
  const data = encoder.encode(text);
  
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
  
  return hashHex;
}

// 使用例
const hash1 = await calculateHash('Hello World');
console.log(hash1);
// a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e

const hash2 = await calculateHash('Hello World!'); // 最後に!を追加
console.log(hash2);
// 7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069

// 1文字違うだけで、全く異なるハッシュ値になる！
```

## 実際の使用例

### ソフトウェアのダウンロード

公式サイトからソフトウェアをダウンロードするとき、こんな表示を見たことはありませんか？

```
ファイル: software.zip
SHA-256: a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e
```

これは、「ダウンロードしたファイルのハッシュ値を計算して、この値と一致するか確認してね」という意味です。
一致すれば、ファイルが改ざんされていないことが保証されます。

### ブロックチェーン

ビットコインなどのブロックチェーンも、ハッシュ関数を使って改ざんを防いでいます。

# 6. 実践：暗号通信を使ってみよう

## Node.jsでHTTPSサーバーを立てる

```javascript
const https = require('https');
const fs = require('fs');

// SSL証明書を読み込む
const options = {
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem')
};

// HTTPSサーバーを作成
https.createServer(options, (req, res) => {
  res.writeHead(200);
  res.end('Hello, Secure World!\n');
}).listen(443);

console.log('HTTPSサーバーが起動しました: https://localhost');
```

## 自己署名証明書の作成（開発用）

```bash
# OpenSSLで秘密鍵を生成
openssl genrsa -out server-key.pem 2048

# 証明書署名要求（CSR）を作成
openssl req -new -key server-key.pem -out server-csr.pem

# 自己署名証明書を作成
openssl x509 -req -in server-csr.pem -signkey server-key.pem -out server-cert.pem -days 365
```

**注意**: 自己署名証明書は開発用です。本番環境では、Let's Encryptなどの正式なCAから証明書を取得してください。

## Let's Encryptで無料のSSL証明書を取得

```bash
# Certbotをインストール（Ubuntu/Debian）
sudo apt-get install certbot

# 証明書を取得
sudo certbot certonly --standalone -d example.com

# 証明書は以下に保存されます
# /etc/letsencrypt/live/example.com/fullchain.pem
# /etc/letsencrypt/live/example.com/privkey.pem
```

## ブラウザでの暗号化通信

```javascript
// Fetch APIは自動的にHTTPSを使用
fetch('https://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    username: 'user',
    password: 'pass'
  })
})
  .then(response => response.json())
  .then(data => console.log(data));

// ブラウザが自動的に：
// 1. TLSハンドシェイクを実行
// 2. サーバーの証明書を検証
// 3. データを暗号化して送信
// 4. 受信したデータを復号化
// これらすべてを裏側でやってくれます！
```

# 7. よくある質問

## Q1: HTTPSなら100%安全？

**A**: いいえ。HTTPSは「通信経路」を守りますが、以下は防げません：

- サーバー自体がハッキングされる
- フィッシングサイト（偽サイトでもHTTPSは使える）
- マルウェアに感染したPC
- ユーザーが自分でパスワードを漏らす

HTTPSは「鍵のかかった郵便配達」であって、「送り先が信頼できるか」は別問題です。

## Q2: 公共Wi-Fiは危険？

**A**: HTTPSを使っていれば、比較的安全です。

昔は「公共Wi-Fiでは絶対にログインするな」と言われましたが、今はほとんどのサイトがHTTPSなので、通信内容は暗号化されています。

ただし、以下には注意：
- HTTPSではないサイトは使わない
- 怪しいWi-Fi（偽のアクセスポイント）に接続しない
- VPNを使うとさらに安全

## Q3: VPNとHTTPSの違いは？

**A**: 守る範囲が違います。

### HTTPS
```
あなた → [暗号化] → サーバー
```
特定のサイトとの通信だけを暗号化

### VPN
```
あなた → [暗号化] → VPNサーバー → インターネット
```
すべての通信を暗号化（どのサイトにアクセスしているかも隠せる）

## Q4: パスワードはどう暗号化されている？

**A**: 2段階で守られています。

1. **通信中**: HTTPSで暗号化
2. **保存時**: ハッシュ化（bcrypt、Argon2など）

```javascript
// サーバー側でのパスワード保存（Node.js + bcrypt）
const bcrypt = require('bcrypt');

// パスワードをハッシュ化して保存
async function savePassword(password) {
  const saltRounds = 10;
  const hashedPassword = await bcrypt.hash(password, saltRounds);
  
  // データベースに保存するのはこのハッシュ値
  // 元のパスワードは保存しない！
  return hashedPassword;
}

// ログイン時の検証
async function verifyPassword(inputPassword, hashedPassword) {
  const match = await bcrypt.compare(inputPassword, hashedPassword);
  return match; // true or false
}
```

# 8. セキュリティのベストプラクティス

## 開発者向け

### 1. 常にHTTPSを使う

```javascript
// HTTPからHTTPSへ強制リダイレクト（Express.js）
app.use((req, res, next) => {
  if (req.header('x-forwarded-proto') !== 'https' && process.env.NODE_ENV === 'production') {
    res.redirect(`https://${req.header('host')}${req.url}`);
  } else {
    next();
  }
});
```

### 2. 最新のTLSバージョンを使う

```javascript
// Node.jsでTLS 1.2以上を強制
const https = require('https');
const tls = require('tls');

const options = {
  minVersion: 'TLSv1.2',
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
};
```

### 3. HSTSヘッダーを設定

```javascript
// HTTP Strict Transport Security
// ブラウザに「このサイトは常にHTTPSで接続すること」を指示
app.use((req, res, next) => {
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  next();
});
```

### 4. 機密情報をログに出力しない

```javascript
// ❌ 悪い例
console.log('ユーザーのパスワード:', password);

// ✅ 良い例
console.log('ユーザー認証成功');
```

## ユーザー向け

### 1. URLを確認する

- `https://` で始まっているか
- ドメイン名が正しいか（`g00gle.com` のような偽物に注意）

### 2. 鍵マークを確認

ブラウザのアドレスバーに🔒マークがあるか確認

### 3. 証明書を確認（重要な取引の場合）

鍵マークをクリックして、証明書の詳細を確認

### 4. 公共のPCでは機密情報を入力しない

暗号化されていても、キーロガー（キー入力を記録するマルウェア）がいるかもしれません

# まとめ

- **暗号通信**は、インターネット上でデータを安全にやり取りするための技術
- **共通鍵暗号**は速いが鍵の配送が課題、**公開鍵暗号**は鍵の配送問題を解決
- **HTTPS**は、SSL/TLSを使ってWebの通信を暗号化している
- **デジタル署名**とハッシュ関数で、改ざんを検出できる
- HTTPSは通信経路を守るが、それだけでは完全な安全は保証されない

暗号通信は、私たちの日常生活に欠かせない技術です。
オンラインショッピング、ネットバンキング、メッセージアプリ…すべてが暗号化によって守られています。

「なんとなく安全」ではなく、「どう守られているか」を理解することで、より安全にインターネットを使えるようになります。

次にブラウザのアドレスバーに🔒マークを見たら、「あ、今TLSハンドシェイクが行われて、AESで暗号化されているんだな」と思い出してみてください。

あなたのデータは、今この瞬間も、暗号技術によって守られています。🔐
