---
title: "パフォーマンスだけを理由に意味論や可読性を捨てるべきではない理由"
emoji: "🧠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "javascript", "typescript", "performance", "設計" ]
published: true
---

パフォーマンスだけを根拠に、意味論や可読性を犠牲にする設計を見かけることがあります。
そのような主張は一見もっともらしく聞こえますが、それが長期的に見て適切な判断かどうかは冷静に見直す必要があります。
この記事では、こうした「気付けない程度の最適化」を理由に、**可読性・意味論を切り捨てることの危険性**について解説します。

:::message
事例はReactやTypeScriptを中心にしていますが、他の言語やフレームワークにも共通する考え方です。
:::

---

## 1. React.Fragment vs div : 速さの幻想 

> React.Fragment(`<></>`)はdivよりもパフォーマンスが悪いから使うべきではない

### 実際はどうか

* React.Fragmentとdivの実行速度の差は、ほとんどのユースケースで体感できないレベルである。
* Reactのバージョンやブラウザによって最適化は変わる。

以前はdivの方がわずかに速いという話もありました。
しかし、**その差は数ミリ秒未満**であり、実際のアプリケーションではほとんど影響しません。
しかも現在はReactの最適化により、Fragmentの方が速いとされています。
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
    <div> {/* 意味のないdiv */}
      <TextField label="ユーザー名" name={user.name} />
      <TextField label="メールアドレス" name={user.email} />
    </div>
  );
}

// ✅ 意図を明確にしたFragment
function UserProfileField({ user }) {
  return (
    <> {/* フィールド要素のグループ化のみが目的 */}
      <TextField label="ユーザー名" name={user.name} />
      <TextField label="メールアドレス" name={user.email} />
    </>
  );
}
```

---

## 2. for文 vs Generator関数：最適化の優先順位を間違えていないか

> Generator関数の方がfor文よりもパフォーマンスが良い

### 本当に効いているのか

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

// ✅ 可読性とシンプルさを優先した例（limit = 100）
function processOrderData(orders: Order[], limit: number): Order[] {
  return orders
    .filter(order => order.status === 'completed')
    .map(order => ({
      ...order,
      amount: Math.round(order.amount / 1.1)
    }))
    .filter(order => order.amount > 10_000)
    .slice(0, limit);
}
```

---

## 3. Barrel Exportは本当にやめるべきか

> index.ts でまとめてエクスポートする barrel export は、パフォーマンスに悪い影響を与えるから避けるべき

この主張の裏にあるのは、**ツリーシェイクの非効率化や、実行時のモジュール解決オーバーヘッド**といった懸念です。

しかし、冷静に整理してみると、その懸念だけで barrel export をやめる理由にはなりません。

### 実行時オーバーヘッドは本当に深刻か

確かに、barrel export によりすべてのモジュールが一度に評価されるケースはあります（静的に解決できない場合など）。ただしこれによるオーバーヘッドは**通常の規模のアプリケーションではごくわずか**であり、**明確な不都合が発生していない限り、避ける理由にはなりません**。

仮に気になるレベルのオーバーヘッドが出ている場合、原因は barrel export ではありません。**モジュールの依存構造や初期化処理の密結合**に問題があると考えられます。

### 可読性・構造の明示性の利点が圧倒的

barrel export の最大のメリットは、**インポートが一箇所に集約され、構造と依存関係が明瞭になること**です。

* インポートがまとめられることで、**ファイルの上部を見るだけでどのモジュールに依存しているかが一目瞭然**
* 複数ファイルから個別にimportするよりも、**import文が簡潔かつ整理され、ノイズが減って可読性が高まる**

これは特にドメインやレイヤー単位の設計において、**意図したまとまりを保つのに有効**であり、チーム開発やリファクタリング時のメンテナンス性にも貢献します。

```typescript
// ❌ 個別インポートで散らかった例
// usecases/CreateUserUseCase.ts
import { ValidationComponent } from '../components/validation/ValidationComponent';
import { NotificationComponent } from '../components/notification/NotificationComponent';
import { LoggingComponent } from '../components/logging/LoggingComponent';
import { CacheComponent } from '../components/cache/CacheComponent';
import { UserService } from '../services/UserService';
import { EmailService } from '../services/EmailService';
import { AuditService } from '../services/AuditService';

// ✅ barrel exportで構造を明確化
// usecases/CreateUserUseCase.ts
import { ValidationComponent, NotificationComponent, LoggingComponent, CacheComponent } from '../components';
import { UserService, EmailService, AuditService } from '../services';
```

### ツリーシェイクの懸念は過去の話

「barrel export はツリーシェイクに不利」という話もありますが、これは **古いバンドラやCJSモジュールが主流だった時代の話**です。

* 現代のバンドラ（Vite、ESBuild、webpack5以降など）はESMベースで静的解析し、**未使用のエクスポートを除去できます**
* TypeScriptとESMを正しく併用していれば、**tree shaking が効かないという状況はほぼ発生しません**

そのため、**この懸念を主な理由として barrel export を避けるのは、技術的にも時代遅れ**な判断と言えるでしょう。

## 4. メモ化はオーバーヘッドがあるから無闇に使わないべきか

> `useMemo` はパフォーマンスが気になるときだけ使えばいい

この考え方は、`useMemo` の役割を半分しか見ていません。`useMemo` はパフォーマンス最適化のためだけにあるのではなく、**値が「派生値」であることを示す意味論的な役割**を持っています。

### `useMemo` は派生値であるという宣言

`useMemo` は、ある値が props や state から計算されて作られる**派生的なデータ**であることをコード上で明示します。
これを使わないということは、その値が「毎回新しく生成される一時的な変数」なのか、それとも「特定の入力から常に同じ結果が導かれるべき値」なのかが曖昧になることを意味します。

パフォーマンスを理由に `useMemo` を省略すると、この「派生値」という重要な意味がコードから失われ、他の開発者がその値を扱う際に誤解を生む可能性があります。
例えば、propsが同じなら再計算の必要がないはずの値が`useMemo`なしで実装されていると、毎回新しいインスタンスが作られてしまい、意図しない再レンダリングを引き起こす原因にもなりえます。

### 意味を伝えるための `useMemo`

`useMemo` を使うことで、その値がどの state や props に依存しているのかが依存配列 `[]` によって明確になります。
これにより、コンポーネント内のデータの流れが追いやすくなり、可読性とメンテナンス性が向上します。

```tsx
// ❌ 派生値であることが分かりにくい例
function ItemSearch({ items }: { items: string[] }) {
  const [query, setQuery] = useState('');

  // queryが変わるたびにフィルタリングが走る
  // `filteredItems`が`query`と`items`からの派生値であることがコード上読み取りにくい
  const filteredItems = items.filter(item => item.toLowerCase().includes(query.toLowerCase()));

  return (
    <div>
      <input type="search" value={query} onChange={e => setQuery(e.target.value)} />
      <ul>
        {filteredItems.map(item => <li key={item}>{item}</li>)}
      </ul>
    </div>
  );
}

// ✅ useMemoで派生値であることを明示
function ItemSearch({ items }: { items: string[] }) {
  const [query, setQuery] = useState('');

  // `query`と`items`から派生した値であることを明示
  // 依存配列のおかげで、いつ再計算されるかが明確になる
  const filteredItems = useMemo(() => {
    return items.filter(item => item.toLowerCase().includes(query.toLowerCase()));
  }, [items, query]);

  return (
    <div>
      <input type="search" value={query} onChange={e => setQuery(e.target.value)} />
      <ul>
        {filteredItems.map(item => <li key={item}>{item}</li>)}
      </ul>
    </div>
  );
}
```

`useMemo` は「パフォーマンスが悪くなったから追加するおまじない」ではありません。
「props や state から値を計算して作り出す」という当たり前の処理を、意味論的に正しく表現するための基本的なツールなのです。
パフォーマンス上のメリットは、その結果として得られる副産物と捉えるべきです。

**参考記事**

https://qiita.com/uhyo/items/59124425ca1dee3da891

---

## 可読性とパフォーマンスはトレードオフではないが、衝突する場合もある

この記事では「パフォーマンスのために意味論や可読性を犠牲にすべきではない」と述べましたが、これはすべてのケースにおいて“きれいなコードが最善”という意味ではありません。
パフォーマンス改善が求められる状況では、あえて直感的でないコードを採用せざるを得ないこともあります。

以下の記事では実際にユーザーの体験に悪影響を与えるパフォーマンス問題を解決した事例が紹介されています。

https://speakerdeck.com/koukimiura/hurontoendonopahuomansutiyuningu

このように、時にはコードの美しさよりもパフォーマンスを選ぶべき場面はあります。

- コードの複雑化が限定的であれば、コメントやユーティリティ関数で意味を補う
- パフォーマンス効果が明確に定量化できる場合のみ、積極的に崩すことを許容する
- ドメインロジックや共通処理は、可読性の優先度が高いため極力維持する

実務では、このようにバランスを取ることも重要です。

---

## まとめ

* 一時的・限定的なパフォーマンス差を理由に意味論や可読性を犠牲にするのは本末転倒
* React.Fragment、Generator関数、barrel export などは、**パフォーマンスより設計意図を優先すべき対象**
* パフォーマンス最適化は、**問題が顕在化してから**で十分
* 将来の自分のために意味あるコードを書いていきましょう
