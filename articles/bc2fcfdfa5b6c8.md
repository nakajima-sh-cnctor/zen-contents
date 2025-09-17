---
title: "Vue3のpropsについて調べてみた"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Vue3","JavaScript","TypeScript","コンポーネント設計","リアクティビティ"]
published: false
---

## 基本的な使い方

`<script setup>`内で`props`を受け取るには、`defineProps`を使用します。これはインポートせずに直接利用できます。

### 配列構文

最もシンプルな方法は、`props`の名前を文字列の配列で指定するものです。

```vue:ChildComponent.vue
<script setup>
// 'message' という名前のpropsを受け取る
const props = defineProps(['message']);

console.log(props.message); // 親から渡された値
</script>

<template>
  <p>{{ message }}</p>
</template>
```

```vue:ParentComponent.vue
<script setup>
import ChildComponent from './ChildComponent.vue';
</script>

<template>
  <ChildComponent message="こんにちは、Vue 3！" />
</template>
```

この例では、親から渡された`"こんにちは、Vue 3！"`という文字列が、子コンポーネントの`message`という`prop`として受け取られます。

-----

## 型を指定する（オブジェクト構文）

より堅牢なコードにするために、各`prop`に型を指定することが推奨されます。

### JavaScriptでの型指定

`String`, `Number`, `Boolean`, `Array`, `Object`, `Date`, `Function`, `Symbol` といったネイティブコンストラクタを型として使用できます。


```vue:ChildComponent.vue
<script setup>
const props = defineProps({
  title: String, // String型
  likes: Number, // Number型
  isPublished: Boolean, // Boolean型
});
</script>

<template>
  <div>
    <h2>{{ title }}</h2>
    <p>いいね数: {{ likes }}</p>
    <p>{{ isPublished ? '公開中' : '非公開' }}</p>
  </div>
</template>
```

-----

### TypeScriptでの型指定

TypeScriptを使っている場合は、ジェネリクス（`< >`）を使ってより厳密に型を定義できます。こちらの方がIDEの補完も効きやすく、開発体験が向上します。

```vue:ChildComponent.vue
<script setup lang="ts">
// 型をインターフェースや型エイリアスとして定義
interface Props {
  title: string;
  likes?: number; // '?' をつけるとオプショナルになる
}

const props = defineProps<Props>();
</script>
```

-----

## Propsのリアクティビティと注意点 ⚠️

`props`はリアクティブなオブジェクトです。親コンポーネントでデータが変更されると、子コンポーネントの`props`も自動的に更新されます。

しかし、よくある間違いとして**propsを分割代入してしまう**ケースがあります。

```javascript
import { defineProps } from 'vue';

const props = defineProps(['message']);

// 🚨 これはNG！ リアクティビティが失われる
let { message } = props;

// 親コンポーネントでmessageが変更されても、このmessage変数の値は更新されない
```

分割代入をすると、その変数はリアクティビティを失い、ただの文字列や数値になってしまいます。

### Vue 3.5以降での改善 🎉

Vue 3.5以降では、propsの分割代入がリアクティビティを維持するようになりました！

```vue:ChildComponent.vue
<script setup>
const props = defineProps(['message']);

// ✅ Vue 3.5以降では、これでリアクティビティが維持される
const { message } = props;

// 親コンポーネントでmessageが変更されると、このmessage変数も自動的に更新される
</script>

<template>
  <p>{{ message }}</p>
</template>
```

この改善により、`toRefs`を使わずに直接分割代入できるようになり、コードがよりシンプルになりました。

### Vue 3.4以前でのリアクティビティを維持する方法

Vue 3.4以前では、`props`の特定のプロパティをリアクティブなまま扱いたい場合は、`toRefs`または`toRef`ユーティリティを使用する必要がありました。

```vue:ChildComponent.vue
<script setup>
import { toRefs } from 'vue';

const props = defineProps({
  message: String,
});

// propsオブジェクト内の各プロパティをリアクティブなrefに変換
const { message } = toRefs(props);

// これでmessageはリアクティブなrefとして扱える
// .valueで値にアクセスする必要がある (template内では不要)
console.log(message.value);
</script>

<template>
  <p>{{ message }}</p>
</template>
```

> **ポイント**: 基本的に`props`はそのまま`props.プロパティ名`の形で使うのが安全です。ロジックの都合でどうしても分割したい場合に`toRefs`を使いましょう。

-----

## デフォルト値とバリデーション

`props`にはデフォルト値を設定したり、必須項目にしたり、独自の検証ルールを追加したりできます。これらはオブジェクト構文で行います。

```vue:UserProfile.vue
<script setup lang="ts">
interface Props {
  name: string;
  age?: number;
  role: 'admin' | 'user' | 'guest';
}

// withDefaults を使ってデフォルト値を設定
const props = withDefaults(defineProps<Props>(), {
  age: 20,
  role: 'user',
});
</script>
```

JavaScriptの場合は、より詳細なバリデーションが可能です。

```javascript:UserProfile.js.vue
<script setup>
defineProps({
  name: {
    type: String,
    required: true, // 必須項目
  },
  age: {
    type: Number,
    default: 20, // デフォルト値
  },
  role: {
    type: String,
    // 値が指定されたいずれかであるかを検証
    validator: (value) => {
      return ['admin', 'user', 'guest'].includes(value);
    },
    default: 'user',
  },
});
</script>
```

これらの機能を活用することで、コンポーネントの再利用性が高まり、予期せぬエラーを防ぐことができます。