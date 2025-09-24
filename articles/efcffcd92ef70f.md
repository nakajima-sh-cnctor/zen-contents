---
title: "Nuxt3 + Firebase + Piniaで作る実践的なチャットアプリの状態管理"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nuxt3", "firebase", "pinia", "typescript", "状態管理"]
published: false
---

## はじめに

Nuxt3とFirebaseを組み合わせたアプリケーション開発において、状態管理は重要な要素の一つです。本記事では、Piniaを使用した実践的な状態管理の実装例を、実際のチャットアプリケーションのコードを交えて解説します。

## プロジェクト構成

今回紹介するプロジェクトは、以下のような構成になっています：

```
app/
├── stores/
│   ├── auth.ts      # 認証状態管理
│   ├── profile.ts   # プロフィール状態管理
│   └── thread.ts    # スレッド状態管理
├── components/
├── pages/
└── composables/
    └── useDateFormat.ts  # 日付フォーマット用のユーティリティ
```

## ComposablesからPiniaへの移行

このプロジェクトでは、当初はNuxt3のComposablesを使用して状態管理を行っていましたが、アプリケーションの複雑さが増すにつれて、より構造化された状態管理が必要になりました。そのため、状態管理部分をPiniaに移行しました。

### 移行前後の比較

**移行前（Composables）:**

- 各コンポーネントで個別に状態を管理
- 状態の共有が困難
- 型安全性の確保が複雑
- 状態の永続化が困難

**移行後（Pinia）:**

- 一元化された状態管理
- ストア間での状態共有が容易
- TypeScriptによる型安全性
- デバッグツールとの統合

### 残したComposables

状態管理以外の機能は、適切にComposablesとして残しています：

```typescript
// useDateFormat.ts - 日付フォーマット用のユーティリティ
export const useDateFormat = () => {
  const formatDate = (
    dateInput: string | Date | undefined | unknown
  ): string => {
    // 日付フォーマット処理
  }

  const formatRelativeTime = (
    dateInput: string | Date | undefined | unknown
  ): string => {
    // 相対時間表示処理
  }

  return {
    formatDate,
    formatDateShort,
    formatRelativeTime,
  }
}
```

このように、**状態管理はPinia**、**再利用可能なロジックはComposables**という使い分けをしています。

## Piniaの基本的な使い方

### 1. 認証ストア（auth.ts）

認証機能を管理するストアの実装例です：

```typescript
import {
  onAuthStateChanged,
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword,
  signOut,
  type User,
  type Auth,
} from 'firebase/auth'

export const useAuthStore = defineStore('auth', () => {
  const { $auth } = useNuxtApp()
  const auth = $auth as Auth | null
  const user = ref<User | null>(null)
  const loading = ref(false)
  let unsubscribe: (() => void) | null = null

  // 認証リスナーを開始
  const startAuthListener = () => {
    loading.value = true
    if (!auth || unsubscribe) {
      loading.value = false
      return
    }

    unsubscribe = onAuthStateChanged(auth, async (currentUser) => {
      user.value = currentUser
      loading.value = false

      // ユーザーが未認証の場合、ログインページにリダイレクト
      if (!currentUser) {
        navigateTo('/login')
      } else {
        // ユーザーが認証された場合、プロフィールを取得
        const profileStore = useProfileStore()
        await profileStore.getProfile(currentUser.uid)
      }
    })
  }

  // ログイン
  const login = async (email: string, password: string) => {
    if (!auth) {
      return {
        user: null,
        error: new Error('Firebase認証が初期化されていません'),
      }
    }
    try {
      const userCredential = await signInWithEmailAndPassword(
        auth,
        email,
        password
      )
      return { user: userCredential.user, error: null }
    } catch (error) {
      return { user: null, error: error as Error }
    }
  }

  // ストアが破棄される際にリスナーをクリーンアップ
  onScopeDispose(() => {
    stopAuthListener()
  })

  return {
    user,
    loading,
    signup,
    login,
    startAuthListener,
    stopAuthListener,
    logout,
    changePassword,
  }
})
```

### 2. プロフィールストア（profile.ts）

ユーザープロフィールを管理するストアです：

```typescript
export interface ProfileData {
  nickname: string
  age: number
  gender: string
  createdAt: Date
  updatedAt: Date
}

export const useProfileStore = defineStore('profile', () => {
  const { $firestore } = useNuxtApp()
  const firestore = $firestore as Firestore | null
  const loading = ref(false)
  const profile = ref<ProfileData | null>(null)

  // プロフィール取得
  const getProfile = async (userId: string) => {
    loading.value = true
    if (!firestore) {
      console.error('Firestoreが初期化されていません')
      return { data: null, error: new Error('Firestoreが初期化されていません') }
    }
    try {
      const profileDoc = await getDoc(doc(firestore, 'profiles', userId))
      if (profileDoc.exists()) {
        const profileData = profileDoc.data() as ProfileData
        profile.value = profileData
        return { data: profileData, error: null }
      } else {
        return { data: null, error: new Error('プロフィールが見つかりません') }
      }
    } catch (error) {
      console.error(error)
      return { data: null, error: error as Error }
    } finally {
      loading.value = false
    }
  }

  return { loading, profile, getProfile, saveProfile, updateProfile }
})
```

### 3. スレッドストア（thread.ts）

チャットスレッドを管理するストアです：

```typescript
export interface Thread {
  id: string
  title: string
  description: string
  authorId: string
  authorName: string
  createdAt: Date
  updatedAt: Date
}

export const useThreadStore = defineStore('thread', () => {
  const { $firestore } = useNuxtApp()
  const firestore = $firestore as Firestore | null
  const threads = ref<Thread[]>([])
  const loading = ref(false)

  // スレッドを作成
  const createThread = async (title: string, description: string) => {
    if (!firestore) {
      return {
        thread: null,
        error: new Error('Firestoreが初期化されていません'),
      }
    }

    const authStore = useAuthStore()
    const profileStore = useProfileStore()

    if (!authStore.user || !profileStore.profile) {
      return {
        thread: null,
        error: new Error(
          'ユーザーが認証されていないか、プロフィールが設定されていません'
        ),
      }
    }

    loading.value = true
    try {
      const threadData = {
        title,
        description,
        authorId: authStore.user.uid,
        authorName: profileStore.profile.nickname,
        createdAt: serverTimestamp(),
        updatedAt: serverTimestamp(),
      }

      const docRef = await addDoc(collection(firestore, 'threads'), threadData)

      const newThread: Thread = {
        id: docRef.id,
        ...threadData,
        createdAt: new Date(),
        updatedAt: new Date(),
      }

      threads.value.unshift(newThread)

      return { thread: newThread, error: null }
    } catch (error) {
      return { thread: null, error: error as Error }
    } finally {
      loading.value = false
    }
  }

  return {
    threads,
    loading,
    createThread,
    getThreads,
  }
})
```

## Piniaの特徴とメリット

### 1. Composition APIスタイル

PiniaはVue 3のComposition APIスタイルでストアを定義できます。これにより、より直感的で理解しやすいコードを書くことができます。

### 2. TypeScriptサポート

PiniaはTypeScriptを完全にサポートしており、型安全性を保ちながら開発できます。

### 3. ストア間の連携

複数のストア間で簡単にデータを共有できます：

```typescript
// authストアからprofileストアを呼び出し
const profileStore = useProfileStore()
await profileStore.getProfile(currentUser.uid)

// threadストアからauthストアとprofileストアを参照
const authStore = useAuthStore()
const profileStore = useProfileStore()
```

### 4. メモリリークの防止

`onScopeDispose`を使用して、ストアが破棄される際にリソースを適切にクリーンアップできます：

```typescript
onScopeDispose(() => {
  stopAuthListener()
})
```

## 実装のポイント

### 1. エラーハンドリング

各メソッドで一貫したエラーハンドリングを行っています：

```typescript
return { user: null, error: error as Error }
```

### 2. ローディング状態の管理

ユーザーエクスペリエンスを向上させるため、適切なローディング状態を管理しています：

```typescript
const loading = ref(false)
```

### 3. Firebaseとの連携

Firebaseの認証状態やFirestoreのデータを適切に管理し、リアルタイムで状態を更新しています。

## ComposablesとPiniaの使い分け

このプロジェクトでは、以下のような使い分けを行っています：

### Piniaを使用する場面

- **グローバルな状態管理**: 認証状態、ユーザープロフィール、アプリケーション全体で共有されるデータ
- **複雑な状態ロジック**: 複数のコンポーネント間での状態の同期が必要な場合
- **永続化が必要な状態**: ページリロード後も保持したい状態

### Composablesを使用する場面

- **再利用可能なロジック**: 日付フォーマット、バリデーション、API呼び出しのヘルパー関数
- **コンポーネント固有の状態**: 特定のコンポーネントでのみ使用される状態
- **純粋な関数**: 副作用のない、入力に対して一意の出力を返す関数

### 移行の判断基準

ComposablesからPiniaへの移行を検討すべきタイミング：

1. **状態の共有が複雑になった時**
2. **複数のコンポーネントで同じ状態を参照する時**
3. **状態の永続化が必要になった時**
4. **デバッグが困難になった時**

## まとめ

Piniaを使用することで、Nuxt3とFirebaseを組み合わせたアプリケーションにおいて、以下のようなメリットを得ることができます：

- **型安全性**: TypeScriptとの完全な統合
- **可読性**: Composition APIスタイルによる直感的なコード
- **保守性**: ストア間の明確な責任分離
- **パフォーマンス**: 必要な部分のみのリアクティブ更新
- **スケーラビリティ**: アプリケーションの成長に合わせた状態管理の拡張

ComposablesとPiniaの適切な使い分けにより、保守性が高く、拡張しやすいアプリケーションを構築できます。今回紹介した実装例を参考に、あなたのプロジェクトでも効果的な状態管理を実現してみてください。

## 参考資料

- [Pinia公式ドキュメント](https://pinia.vuejs.org/)
- [Nuxt3公式ドキュメント](https://nuxt.com/)
- [Firebase公式ドキュメント](https://firebase.google.com/docs)
