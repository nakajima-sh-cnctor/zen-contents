---
title: "プロフィール作成画面の作成"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "nuxt", "firebase", "vuetify", "typescript"]
published: false
---

# はじめに

この記事では、Vue.js + Nuxt.js + Firebase + Vuetifyを使用してプロフィール作成画面を実装する方法を詳しく解説します。ユーザーがニックネーム、年齢、性別を入力し、Firebase Firestoreに保存する機能を構築していきます。

## 実装する機能

- プロフィール情報の入力フォーム（ニックネーム、年齢、性別）
- リアルタイムバリデーション
- Firebase Firestoreへのデータ保存
- エラーハンドリング
- ローディング状態の管理

## 技術スタック

- **フロントエンド**: Vue.js 3 + Nuxt.js 3
- **UI フレームワーク**: Vuetify 3
- **データベース**: Firebase Firestore
- **言語**: TypeScript
- **状態管理**: Vue Composition API

## 要件

- ニックネーム
- 年齢
- 性別

上記入力項目を登録する画面を作成
FirebaseのFirestore Databaseを使いプロフィールを登録します

## アーキテクチャ概要

この実装では、以下のような構成でプロフィール作成機能を構築します：

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Vue Page      │    │   Component     │    │   Composable    │
│ (create-profile)│───▶│ (ProfileForm)   │───▶│ (useProfile)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       ▼
         │                       │              ┌─────────────────┐
         │                       │              │ Firebase        │
         │                       │              │ Firestore       │
         │                       │              └─────────────────┘
         ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│   Vuetify UI    │    │  Validation     │
│   Components    │    │  Rules          │
└─────────────────┘    └─────────────────┘
```

### コンポーネント設計のポイント

1. **単一責任の原則**: 各コンポーネントは明確な役割を持つ
2. **再利用性**: ProfileFormコンポーネントは作成・編集両方で使用可能
3. **疎結合**: コンポーネント間の依存関係を最小限に抑制
4. **型安全性**: TypeScriptによる型定義で実行時エラーを防止

## Firebase Firestore Databaseとは

Firestore Databaseは、Googleが提供するNoSQLドキュメントデータベースです。リアルタイム同期、オフライン対応、スケーラビリティなどの特徴があります。

### 主な特徴

- **NoSQLドキュメントデータベース**: 従来のリレーショナルデータベースとは異なり、JSONライクなドキュメント形式でデータを保存
- **リアルタイム同期**: データの変更が即座にクライアントに反映される
- **オフライン対応**: ネットワーク接続がなくてもデータの読み書きが可能
- **自動スケーリング**: ユーザー数の増加に応じて自動的にスケールする
- **セキュリティルール**: データベースレベルでのアクセス制御が可能

### データ構造

Firestoreでは以下の階層構造でデータを管理します：

```
データベース
└── コレクション（例：users）
    └── ドキュメント（例：user123）
        └── フィールド（例：nickname, age, gender）
```

### プロフィールデータの保存例

```typescript
// Firestoreにプロフィールデータを保存
const saveProfile = async (profileData: {
  nickname: string;
  age: number;
  gender: string;
}) => {
  try {
    await addDoc(collection(db, 'profiles'), {
      nickname: profileData.nickname,
      age: profileData.age,
      gender: profileData.gender,
      createdAt: serverTimestamp(),
    });
    console.log('プロフィールが正常に保存されました');
  } catch (error) {
    console.error('プロフィールの保存に失敗しました:', error);
  }
};
```

## バリデーションルール

各入力項目には以下のバリデーションルールを設定しています：

### ニックネーム
- 必須入力チェック

### 年齢
- 必須入力チェック
- 数字のみの入力チェック
- 0〜150の範囲内チェック

### 性別
- 必須入力チェック

```typescript
const nicknameRules = [(v: string) => !!v || 'ニックネームは必須です']
const ageRules = [
  (v: string) => !!v || '年齢は必須です',
  (v: string) => /^\d+$/.test(v) || '年齢は数字で入力してください',
  (v: string) => {
    const num = parseInt(v)
    return (num >= 0 && num <= 150) || '年齢は0〜150の範囲で入力してください'
  },
]
const genderRules = [(v: string) => !!v || '性別は必須です']
```


## 入力項目のコンポーネントの作成

プロフィール作成画面の核となるフォームコンポーネントを作成します。このコンポーネントは以下の特徴を持ちます：

### 設計思想

- **再利用性**: プロフィール作成・編集の両方で使用可能
- **バリデーション**: リアルタイムでの入力値チェック
- **アクセシビリティ**: Vuetifyのアクセシブルなコンポーネントを活用
- **型安全性**: TypeScriptによる厳密な型定義

```vue:app/components/ProfileForm.vue
<script setup lang="ts">
interface Props {
  initialNickname?: string
  initialAge?: string
  initialGender?: string
  submitButtonText?: string
  isSubmitting?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  initialNickname: '',
  initialAge: '',
  initialGender: '',
  submitButtonText: '作成',
  isSubmitting: false,
})

const form = ref()

const nickname = ref(props.initialNickname)
const age = ref(props.initialAge)
const gender = ref(props.initialGender)

const nicknameRules = [(v: string) => !!v || 'ニックネームは必須です']
const ageRules = [
  (v: string) => !!v || '年齢は必須です',
  (v: string) => /^\d+$/.test(v) || '年齢は数字で入力してください',
  (v: string) => {
    const num = parseInt(v)
    return (num >= 0 && num <= 150) || '年齢は0〜150の範囲で入力してください'
  },
]
const genderRules = [(v: string) => !!v || '性別は必須です']

// 親コンポーネントへのイベント送信
const emit = defineEmits(['submit'])

const submit = async () => {
  if (form.value) {
    const { valid } = await form.value.validate()
    if (!valid) return
  }
  emit('submit', {
    nickname: nickname.value,
    age: age.value,
    gender: gender.value,
  })
}
</script>

<template>
  <v-form ref="form" @submit.prevent="submit">
    <v-text-field
      v-model="nickname"
      color="primary"
      variant="underlined"
      label="ニックネーム"
      :rules="nicknameRules"
    />
    <v-text-field
      v-model="age"
      color="primary"
      variant="underlined"
      label="年齢"
      :rules="ageRules"
    />
    <v-select
      v-model="gender"
      color="primary"
      variant="underlined"
      label="性別"
      :rules="genderRules"
      :items="['男性', '女性', 'その他']"
    />

    <v-btn
      color="primary"
      class="mt-4"
      type="submit"
      block
      :loading="props.isSubmitting"
      :disabled="props.isSubmitting"
      >{{ props.submitButtonText }}</v-btn
    >
  </v-form>
</template>
```

### 主要な処理の流れ

#### 1. Props定義
```typescript
interface Props {
  initialNickname?: string
  initialAge?: string
  initialGender?: string
  submitButtonText?: string
  isSubmitting?: boolean
}
```
- 親コンポーネントから受け取るプロパティを定義
- 初期値やボタンテキスト、送信状態などを設定可能

#### 2. リアクティブデータの初期化
```typescript
const nickname = ref(props.initialNickname)
const age = ref(props.initialAge)
const gender = ref(props.initialGender)
```
- 各入力フィールドの値をリアクティブに管理
- 親から渡された初期値で初期化

#### 3. バリデーションルールの定義
- **ニックネーム**: 必須入力チェック
- **年齢**: 必須入力、数字のみ、0〜150の範囲内
- **性別**: 必須入力チェック

#### 4. フォーム送信処理
```typescript
const submit = async () => {
  if (form.value) {
    const { valid } = await form.value.validate()
    if (!valid) return
  }
  emit('submit', {
    nickname: nickname.value,
    age: age.value,
    gender: gender.value,
  })
}
```
- フォームのバリデーションを実行
- バリデーションが通った場合のみ、親コンポーネントにデータを送信

#### 5. テンプレート部分
- Vuetifyのコンポーネントを使用したUI
- `v-form`でフォーム全体を管理
- 各入力フィールドにバリデーションルールを適用
- 送信ボタンは送信中状態に応じてローディング表示と無効化

### 特徴

1. **再利用性**: Propsで初期値やボタンテキストを変更可能
2. **バリデーション**: リアルタイムで入力値の妥当性をチェック
3. **状態管理**: 送信中の状態を適切に管理
4. **イベント駆動**: 親コンポーネントとの疎結合な設計

このコンポーネントは、プロフィール作成・編集の両方の場面で使用できる汎用的な設計になっています。

### 実装のポイント

#### 1. Props設計の工夫
```typescript
interface Props {
  initialNickname?: string
  initialAge?: string
  initialGender?: string
  submitButtonText?: string
  isSubmitting?: boolean
}
```
- すべてのプロパティをオプショナルにすることで、柔軟な使用を可能にする
- `submitButtonText`により、作成・更新でボタンテキストを変更可能
- `isSubmitting`で送信状態を外部から制御

#### 2. バリデーション戦略
- **リアルタイムバリデーション**: ユーザーが入力中に即座にフィードバック
- **複数ルール**: 各フィールドに複数のバリデーションルールを適用
- **ユーザーフレンドリー**: 分かりやすいエラーメッセージを表示

#### 3. イベント駆動設計
```typescript
const emit = defineEmits(['submit'])
```
- 親コンポーネントとの疎結合を実現
- フォームの送信処理は親に委譲
- コンポーネントの再利用性を向上

## プロフィールデータをFirestoreに保存する

プロフィールデータをFirestoreに保存するためのComposableを作成します。このComposableは、データの保存処理とローディング状態を管理します。

```ts:app/composables/useProfile.ts
import { doc, setDoc, type Firestore } from 'firebase/firestore'
import { useNuxtApp } from 'nuxt/app'

export interface ProfileData {
  nickname: string
  age: number
  gender: string
  createdAt: Date
  updatedAt: Date
}

export const useProfile = () => {
  const { $firestore } = useNuxtApp()
  const firestore = $firestore as Firestore | null
  const loading = ref(false)

  // プロフィール保存
  const saveProfile = async (
    nickname: string,
    age: string,
    gender: string,
    userId: string
  ) => {
    loading.value = true
    if (!firestore) {
      return {
        error: new Error('Firestoreが初期化されていません'),
      }
    }
    try {
      const profileData: ProfileData = {
        nickname,
        age: parseInt(age),
        gender,
        createdAt: new Date(),
        updatedAt: new Date(),
      }
      await setDoc(doc(firestore, 'profiles', userId), profileData)
      return { error: null }
    } catch (error) {
      return { error: error as Error }
    } finally {
      loading.value = false
    }
  }

  return {
    saveProfile,
    loading: readonly(loading),
  }
}
```

### 主要な処理の流れ

#### 1. インポートとインターフェース定義
```typescript
import { doc, setDoc, type Firestore } from 'firebase/firestore'
import { useNuxtApp } from 'nuxt/app'

export interface ProfileData {
  nickname: string
  age: number
  gender: string
  createdAt: Date
  updatedAt: Date
}
```
- **Firebase Firestore**: `doc`と`setDoc`をインポートしてドキュメントの作成・更新を行う
- **Nuxt App**: `useNuxtApp`でNuxtアプリケーションのインスタンスにアクセス
- **ProfileData**: プロフィールデータの型定義。作成日時と更新日時も含む

#### 2. Composableの初期化
```typescript
export const useProfile = () => {
  const { $firestore } = useNuxtApp()
  const firestore = $firestore as Firestore | null
  const loading = ref(false)
```
- **Firestoreインスタンス**: NuxtアプリからFirestoreインスタンスを取得
- **ローディング状態**: 非同期処理中の状態を管理するリアクティブ変数

#### 3. プロフィール保存処理
```typescript
const saveProfile = async (
  nickname: string,
  age: string,
  gender: string,
  userId: string
) => {
  loading.value = true
  if (!firestore) {
    return {
      error: new Error('Firestoreが初期化されていません'),
    }
  }
  try {
    const profileData: ProfileData = {
      nickname,
      age: parseInt(age),
      gender,
      createdAt: new Date(),
      updatedAt: new Date(),
    }
    await setDoc(doc(firestore, 'profiles', userId), profileData)
    return { error: null }
  } catch (error) {
    return { error: error as Error }
  } finally {
    loading.value = false
  }
}
```

**処理の詳細：**

- **パラメータ**: ニックネーム、年齢（文字列）、性別、ユーザーIDを受け取る
- **ローディング開始**: 処理開始時にローディング状態をtrueに設定
- **Firestoreチェック**: Firestoreが初期化されているかチェック
- **データ変換**: 年齢を文字列から数値に変換（`parseInt(age)`）
- **タイムスタンプ**: 作成日時と更新日時を現在時刻で設定
- **Firestore保存**: `setDoc`で`profiles`コレクションの`userId`ドキュメントに保存
- **エラーハンドリング**: エラーが発生した場合はエラーオブジェクトを返す
- **ローディング終了**: 処理完了時にローディング状態をfalseに設定

#### 4. 戻り値
```typescript
return {
  saveProfile,
  loading: readonly(loading),
}
```
- **saveProfile**: プロフィール保存関数を公開
- **loading**: 読み取り専用のローディング状態を公開

### 特徴

1. **エラーハンドリング**: Firestoreの初期化チェックとtry-catch文でエラーを適切に処理
2. **型安全性**: TypeScriptの型定義により、データの整合性を保証
3. **状態管理**: ローディング状態をリアクティブに管理
4. **再利用性**: Composableとして実装することで、複数のコンポーネントで利用可能
5. **Firestore最適化**: `setDoc`を使用してドキュメントの作成・更新を効率的に実行

### 使用例

```typescript
// コンポーネント内での使用
const { saveProfile, loading } = useProfile()

const handleSubmit = async (formData: any) => {
  const result = await saveProfile(
    formData.nickname,
    formData.age,
    formData.gender,
    currentUser.value?.uid
  )
  
  if (result.error) {
    console.error('保存に失敗しました:', result.error)
  } else {
    console.log('プロフィールが正常に保存されました')
  }
}
```

このComposableにより、プロフィールデータの保存処理が簡潔かつ安全に実装できます。

### Composable設計の利点

#### 1. 関心の分離
- **データアクセス層**: Firestoreとの通信処理を分離
- **ビジネスロジック**: プロフィール関連の処理を集約
- **状態管理**: ローディング状態を一元管理

#### 2. テスト容易性
```typescript
// テスト例
const mockFirestore = createMockFirestore()
const { saveProfile } = useProfile(mockFirestore)
```
- モックオブジェクトを使用した単体テストが容易
- ビジネスロジックとデータアクセス層の分離により、テストの範囲を明確化

#### 3. 再利用性
- 複数のコンポーネントで同じロジックを共有
- プロフィール関連の機能拡張時の影響範囲を最小化

## プロフィール作成ページ

```vue:app/pages/home/create-profile.vue
<script setup lang="ts">
// プロフィール作成フォームコンポーネントをインポート
import ProfileForm from '~/components/ProfileForm.vue'

// プロフィール保存機能とローディング状態を取得
const { saveProfile, loading } = useProfile()
// 認証されたユーザー情報を取得
const { user } = useAuth()

// エラーメッセージを管理するリアクティブ変数
const error = ref('')

// プロフィール送信処理
const handleProfileSubmit = async (data: {
  nickname: string
  age: string
  gender: string
}) => {
  // プロフィール保存を実行（ユーザーIDも含める）
  const { error: saveProfileError } = await saveProfile(
    data.nickname,
    data.age,
    data.gender,
    user.value?.uid ?? ''
  )

  // エラーが発生した場合はエラーメッセージを表示して処理を終了
  if (saveProfileError) {
    error.value = saveProfileError.message
    return
  }

  // プロフィール保存成功後、ホームページにリダイレクト
  await navigateTo('/home')
}
</script>

<template>
  <!-- プロフィール作成用のカードコンポーネント -->
  <v-card class="mx-auto my-8" max-width="500">
    <v-card-item>
      <v-card-title>プロフィール作成</v-card-title>
    </v-card-item>
    <v-card-item>
      <!-- エラーが発生した場合に表示するアラート -->
      <v-alert
        v-if="error"
        type="error"
        :text="error"
        variant="tonal"
        class="mb-4"
      />
      <!-- プロフィールフォームコンポーネント -->
      <!-- loading状態を渡し、submitイベントをハンドル -->
      <ProfileForm :is-submitting="loading" @submit="handleProfileSubmit" />
    </v-card-item>
  </v-card>
</template>
```

### ページコンポーネントの設計思想

#### 1. 責任の明確化
- **ページコンポーネント**: ルーティング、認証、エラーハンドリングを担当
- **フォームコンポーネント**: UI表示とバリデーションを担当
- **Composable**: データアクセスとビジネスロジックを担当

#### 2. エラーハンドリング戦略
```typescript
const error = ref('')

if (saveProfileError) {
  error.value = saveProfileError.message
  return
}
```
- **ユーザーフレンドリー**: 技術的なエラーを分かりやすいメッセージに変換
- **視覚的フィードバック**: Vuetifyのアラートコンポーネントでエラーを表示
- **処理の継続**: エラー発生時は適切に処理を停止

#### 3. ナビゲーション管理
```typescript
await navigateTo('/home')
```
- **成功時の遷移**: プロフィール保存成功後に適切なページへリダイレクト
- **非同期処理**: `await`を使用して確実に遷移を実行

## ベストプラクティスと注意点

### セキュリティ考慮事項

#### 1. Firestoreセキュリティルール
```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /profiles/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```
- **認証チェック**: ログインユーザーのみアクセス可能
- **所有者チェック**: 自分のプロフィールのみ編集可能
- **データ検証**: 入力データの形式をサーバーサイドで検証

#### 2. クライアントサイドバリデーションの限界
```typescript
// クライアントサイドバリデーションは補助的なもの
const ageRules = [
  (v: string) => !!v || '年齢は必須です',
  (v: string) => /^\d+$/.test(v) || '年齢は数字で入力してください',
  // サーバーサイドでも同様の検証が必要
]
```
- **UX向上**: リアルタイムフィードバックでユーザビリティを向上
- **セキュリティ**: サーバーサイドバリデーションは必須
- **二重チェック**: クライアント・サーバー両方で検証を実装

### パフォーマンス最適化

#### 1. コンポーネントの最適化
```typescript
// 不要な再レンダリングを防ぐ
const nickname = ref(props.initialNickname)
const age = ref(props.initialAge)
const gender = ref(props.initialGender)

// 計算プロパティでメモ化
const isValidForm = computed(() => {
  return nickname.value && age.value && gender.value
})
```

#### 2. Firestoreクエリの最適化
```typescript
// 必要なフィールドのみ取得
const profileData: ProfileData = {
  nickname,
  age: parseInt(age),
  gender,
  createdAt: new Date(),
  updatedAt: new Date(),
}
// インデックスを適切に設定
await setDoc(doc(firestore, 'profiles', userId), profileData)
```

### エラーハンドリングのベストプラクティス

#### 1. 階層的エラーハンドリング
```typescript
try {
  const { error: saveProfileError } = await saveProfile(...)
  if (saveProfileError) {
    // ビジネスロジックエラー
    error.value = saveProfileError.message
    return
  }
} catch (error) {
  // 予期しないエラー
  console.error('予期しないエラーが発生しました:', error)
  error.value = 'システムエラーが発生しました。しばらく時間をおいて再度お試しください。'
}
```

#### 2. ユーザーフレンドリーなエラーメッセージ
```typescript
const getErrorMessage = (error: Error): string => {
  switch (error.code) {
    case 'permission-denied':
      return 'この操作を実行する権限がありません。'
    case 'unavailable':
      return 'サービスが一時的に利用できません。しばらく時間をおいて再度お試しください。'
    default:
      return 'エラーが発生しました。しばらく時間をおいて再度お試しください。'
  }
}
```

### アクセシビリティの考慮

#### 1. セマンティックHTML
```vue
<template>
  <v-form ref="form" @submit.prevent="submit" role="form" aria-label="プロフィール作成フォーム">
    <v-text-field
      v-model="nickname"
      label="ニックネーム"
      :rules="nicknameRules"
      aria-describedby="nickname-error"
    />
    <!-- エラーメッセージに適切なIDを設定 -->
  </v-form>
</template>
```

#### 2. キーボードナビゲーション
- Vuetifyコンポーネントは標準でキーボードナビゲーションをサポート
- フォーカス管理とタブオーダーを適切に設定
- スクリーンリーダー対応のaria属性を活用

## まとめ

この記事では、Vue.js + Nuxt.js + Firebase + Vuetifyを使用したプロフィール作成画面の実装について詳しく解説しました。

### 実装のポイント

1. **コンポーネント設計**: 再利用性と保守性を重視した設計
2. **型安全性**: TypeScriptによる厳密な型定義
3. **エラーハンドリング**: ユーザーフレンドリーなエラー処理
4. **セキュリティ**: Firestoreセキュリティルールの適切な設定
5. **アクセシビリティ**: すべてのユーザーが利用可能なUI設計

### 今後の拡張可能性

- **プロフィール画像アップロード**: Firebase Storageとの連携
- **プロフィール編集機能**: 既存データの更新処理
- **バリデーション強化**: より詳細な入力値チェック
- **国際化対応**: 多言語サポートの実装
- **オフライン対応**: PWA機能の追加

この実装をベースに、より高度な機能を追加していくことで、本格的なWebアプリケーションを構築できます。