---
title: "Nuxt3 + Firebaseでプロフィール編集画面を作成する"
emoji: "👤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nuxt3", "firebase", "vue3", "typescript"]
published: false
---

## はじめに

Webアプリケーションでユーザープロフィールの編集機能を実装する際、Nuxt3とFirebaseを組み合わせることで、効率的で安全なプロフィール管理システムを構築できます。

この記事では、以下の機能を持つプロフィール編集画面の実装方法を紹介します：

- プロフィール情報の取得・保存・更新
- リアルタイムでのローディング状態管理
- エラーハンドリング
- TypeScriptによる型安全性

## 実装の概要

プロフィール編集機能は以下の2つの主要なコンポーネントで構成されています：

1. **useProfile Composable**: Firebase Firestoreとの通信を担当
2. **プロフィール編集ページ**: ユーザーインターフェースを提供

## 1. useProfile Composable

Firebase Firestoreとの通信を管理するComposableを作成します。

```ts:app/composables/useProfile.ts
import {
  doc,
  setDoc,
  getDoc,
  updateDoc,
  type Firestore,
} from 'firebase/firestore'
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
  ...省略
  // プロフィール更新
  const updateProfile = async (
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
      const profileData = {
        nickname,
        age: parseInt(age),
        gender,
        updatedAt: new Date(),
      }
      await updateDoc(doc(firestore, 'profiles', userId), profileData)
      return { error: null }
    } catch (error) {
      return { error: error as Error }
    } finally {
      loading.value = false
    }
  }

  return {
    ...省略
    updateProfile,
    loading: readonly(loading),
  }
}
```

### コードの解説

**ProfileDataインターフェース**
- プロフィールデータの型定義を行います
- `nickname`、`age`、`gender`などの基本情報に加え、作成日時と更新日時を管理

**主要な関数**
- `saveProfile`: 新規プロフィールの保存
- `getProfile`: 既存プロフィールの取得
- `updateProfile`: プロフィール情報の更新

**エラーハンドリング**
- Firestoreの初期化チェック
- 非同期処理でのエラーキャッチ
- 適切なエラーメッセージの返却

## 2. プロフィール編集ページ

ユーザーインターフェースを提供するページコンポーネントです。

```vue:app/pages/home/edit-profile.vue
<script setup lang="ts">
import ProfileForm from '~/components/ProfileForm.vue'
import type { ProfileData } from '~/composables/useProfile'

const { getProfile, updateProfile, loading } = useProfile()
const { user } = useAuth()

const error = ref('')
const profile = ref<ProfileData | null>(null)

// ユーザー情報が取得できた時にプロフィール情報を取得
const loadProfile = async () => {
  if (!user.value?.uid) {
    return
  }

  const { data, error: getProfileError } = await getProfile(user.value.uid)

  if (getProfileError) {
    error.value = getProfileError.message
    return
  }

  if (!data) {
    error.value = 'プロフィールが見つかりません'
    return
  }

  profile.value = data
}

// ユーザー情報の変更を監視
watch(
  user,
  (newUser) => {
    if (newUser?.uid) {
      loadProfile()
    }
  },
  { immediate: true }
)

const handleProfileSubmit = async (data: {
  nickname: string
  age: string
  gender: string
}) => {
  if (!user.value?.uid) {
    error.value = 'ユーザー情報が取得できません'
    return
  }

  const { error: updateProfileError } = await updateProfile(
    data.nickname,
    data.age,
    data.gender,
    user.value.uid
  )

  if (updateProfileError) {
    error.value = updateProfileError.message
    return
  }

  // プロフィール更新成功後、ホームページにリダイレクト
  await navigateTo('/home')
}
</script>

<template>
  <v-card class="mx-auto my-8" max-width="500">
    <v-card-item>
      <v-card-title>プロフィール編集</v-card-title>
    </v-card-item>
    <v-card-item>
      <v-alert
        v-if="error"
        type="error"
        :text="error"
        variant="tonal"
        class="mb-4"
      />

      <!-- プロフィール情報が読み込まれている場合のみフォームを表示 -->
      <ProfileForm
        v-if="profile"
        :initial-nickname="profile.nickname"
        :initial-age="profile.age.toString()"
        :initial-gender="profile.gender"
        submit-button-text="更新"
        :is-submitting="loading"
        @submit="handleProfileSubmit"
      />

      <!-- ローディング中の場合 -->
      <div v-else-if="loading" class="text-center py-8">
        <v-progress-circular indeterminate color="primary" />
        <p class="mt-4">プロフィール情報を読み込み中...</p>
      </div>
    </v-card-item>
  </v-card>
</template>
```

### ページコンポーネントの解説

**データ管理**
- `profile`: 現在のプロフィール情報を保持
- `error`: エラーメッセージの管理
- `loading`: ローディング状態の管理

**ライフサイクル管理**
- `watch`を使用してユーザー情報の変更を監視
- ユーザーが認証されたタイミングでプロフィール情報を自動取得

**フォーム処理**
- `handleProfileSubmit`: フォーム送信時の処理
- 更新成功後はホームページにリダイレクト

**UI/UX**
- ローディング中の表示
- エラー時のアラート表示
- プロフィール情報が読み込まれるまでフォームを非表示

## 3. 必要な依存関係

この実装には以下のパッケージが必要です：

```json:package.json
{
  "dependencies": {
    "nuxt": "^3.0.0",
    "firebase": "^10.0.0",
    "vue": "^3.0.0"
  }
}
```

## まとめ

この記事では、Nuxt3とFirebaseを使用したプロフィール編集機能の実装方法を紹介しました。

**実装のポイント**
- Composableパターンによるロジックの分離
- TypeScriptによる型安全性の確保
- 適切なエラーハンドリングとローディング状態管理
- ユーザーフレンドリーなUI/UX

この実装を参考に、より複雑なプロフィール機能（アバター画像のアップロード、複数のプロフィール項目など）への拡張も可能です。

Firebaseのセキュリティルールの設定も忘れずに行い、適切な認証と認可を実装してください。