---
title: "パフォーマンスだけを理由に意味論や可読性を捨てるべきではない理由"
emoji: "🧠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "javascript", "performance", "設計", "可読性"]
published: true
---

パフォーマンスだけを根拠に、意味論や可読性を犠牲にする設計を見かけることがあります。
そのような主張は一見もっともらしく聞こえますが、それが長期的に見て適切な判断かどうかは冷静に見直す必要があります。
この記事では、こうした「気付けない程度の最適化」を理由に、**可読性・意味論を切り捨てることの危険性**について解説します。

---

## 1. React.Fragment vs div : 速さの幻想 

> React.Fragment(`<></>`)はdivよりもパフォーマンスが悪いから使うべきではない

### 実際はどうか？

* React.Fragmentとdivの実行速度の差は、ほとんどのユースケースで体感できないレベルです。
* Reactのバージョンやブラウザによって最適化は変わります。

以前はdivの方がわずかに速いという話もありました。
しかし、**その差は数ミリ秒以下**であり、実際のアプリケーションではほとんど影響しません。
ちなみに現在はReactの最適化により、Fragmentの方が速いらしいです。
つまり、**特定の実装・特定の環境における一時的な差を根拠にするのは危険**です。

### Fragmentを選ぶべき理由

React.Fragmentは「要素をグループ化したいが、DOMには何も追加したくない」という**明確な意図**を持っています。
これをわざわざdivに変えることは、意味のない余計なDOMノードを追加することになり、CSS設計やアクセシビリティの面でも悪影響を及ぼすことになります。
要素をグループ化したいだけであれば、Fragmentを使いましょう。

```tsx
import { TextField } from './ui/TextField';

// ❌ パフォーマンスを理由にdivを使う
function UserProfileField({ user }) {
  return (
    <div> {/* 意味のないwrapper div */}
      <TextField label="ユーザー名" value={user.name} />
      <TextField label="メールアドレス" value={user.email} />
    </div>
  );
}

// ✅ 意図を明確にしたFragment
function UserProfileField({ user }) {
  return (
    <> {/* フィールド要素のグループ化のみが目的 */}
      <TextField label="ユーザー名" value={user.name} />
      <TextField label="メールアドレス" value={user.email} />
    </>
  );
}
```

---

## 2. for文 vs Generator関数：最適化の優先順位を間違えていないか？

> Generator関数の方がfor文よりもパフォーマンスが良い

### それ、本当に効いてる？

* Generator関数は遅延評価によるメモリ効率の利点がありますが、少量のデータ処理においてはパフォーマンスに差はほぼ出ません。
* 可読性やメンテナンス性を損なってまで使うべきものではありません。

### 不都合がないなら最適化しない

確かに、Generator関数がループ処理で優れているケースはあります。
しかし、実行時間の差が体感できるほどになるには、非常に大量のデータを扱うケースに限られます。

> 早すぎる最適化はあらゆる悪の根源である

このような言葉があるように、パフォーマンス改善は、**不都合が実際に発生してからで十分**です。
プロファイルを取っても差がほぼ検出できないような改善は、"気づかれない最適化"にすぎず、むしろ可読性を犠牲にすることで負債になります。

```ts
// ❌ パフォーマンスを理由に可読性を犠牲にした例
function* filterLargeCompletedOrders(orders: Iterable<Order>, limit: number): Generator<Order> {
  let count = 0
  for (const order of orders) {
    if (order.status !== 'completed') continue

    const taxExcluded = Math.round(order.amount / 1.1)
    if (taxExcluded <= 10_000) continue

    yield { ...order, amount: taxExcluded }

    count++
    if (count >= limit) return
  }
}

// ✅ 可読性とシンプルさを優先した例
function processOrderData(orders): Order[] {
  return orders
    .filter(order => order.status === 'completed')
    .map(order => ({
      ...order,
      amount: Math.round(order.amount / 1.1)
    }))
    .filter(order => order.amount > 10_000)
    .slice(0, 100);
}
```

---

## 3. Barrel Exportは本当にやめるべきか？

> index.ts でまとめてエクスポートする barrel export は、パフォーマンスに悪い影響を与えるから避けるべき

この主張の裏にあるのは、**ツリーシェイクの非効率化や、実行時のモジュール解決オーバーヘッド**といった懸念です。

しかし、冷静に整理してみると、その懸念だけで barrel export をやめる理由にはなりません。

### 実行時オーバーヘッドは本当に深刻か？

確かに、barrel export によりすべてのモジュールが一度に評価されるケースはあります（静的に解決できない場合など）。ただしこれによるオーバーヘッドは**通常の規模のアプリケーションではごくわずか**であり、**明確な不都合が発生していない限り、避ける理由にはなりません**。

仮に気になるレベルのオーバーヘッドが出ている場合、それは barrel export のせいというより、**モジュールの依存構造や初期化処理が密結合になっていること自体が問題**である可能性が高いです。

### 可読性・構造の明示性の利点が圧倒的

barrel export の最大のメリットは、**インポートが一箇所に集約され、構造と依存関係が明瞭になること**です。

* インポートがまとめられることで、**ファイルの上部を見るだけでどのモジュールに依存しているかが一目瞭然**
* 複数ファイルから個別にimportするよりも、**import文が簡潔かつ整理され、ノイズが減って可読性が高まる**

これは特にドメインやレイヤー単位の設計において、**意図したまとまりを保つのに有効**であり、チーム開発やリファクタリング時のメンテナンス性にも貢献します。

```typescript
// ❌ 個別インポートで散らかった例
import { GetUserProfileUseCase } from '../domain/usecases/user/GetUserProfileUseCase';
import { ValidateUserUseCase } from '../domain/usecases/user/ValidateUserUseCase';
import { SendNotificationUseCase } from '../domain/usecases/notification/SendNotificationUseCase';
import { UserRepository } from '../infrastructure/repositories/UserRepository';
import { NotificationRepository } from '../infrastructure/repositories/NotificationRepository';
import { EmailService } from '../infrastructure/services/EmailService';

// ✅ barrel exportでレイヤーを明確化
import { GetUserProfileUseCase, ValidateUserUseCase } from '../domain/usecases/user';
import { SendNotificationUseCase } from '../domain/usecases/notification';
import { UserRepository, NotificationRepository } from '../infrastructure/repositories';
import { EmailService } from '../infrastructure/services';
```

### ツリーシェイクの懸念は過去の話

「barrel export はツリーシェイクに不利」という話もありますが、これは **古いバンドラやCJSモジュールが主流だった時代の話**です。

* 現代のバンドラ（Vite、ESBuild、webpack5以降など）はESMベースで静的解析を行い、**未使用のエクスポートを除去できます**
* TypeScriptとESMを正しく併用していれば、**tree shaking が効かないという状況はほぼ発生しません**

そのため、**この懸念を主な理由として barrel export を避けるのは、技術的にも時代遅れ**な判断と言えるでしょう。

---

## まとめ

* 一時的・限定的なパフォーマンス差を理由に意味論や可読性を犠牲にするのは本末転倒
* React.Fragment、Generator関数、barrel export などは、**パフォーマンスより設計意図を優先すべき対象**
* パフォーマンス最適化は、**問題が顕在化してから**で十分
* 将来の自分のために意味あるコードを書いていきましょう！
