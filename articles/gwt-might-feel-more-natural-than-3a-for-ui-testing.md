---
title: 'UI のテストは AAA パターンより Given-When-Then パターンの方がしっくりくるかもしれない'
emoji: '📝'
type: 'tech'
topics: ['test', 'jest', 'react', 'testinglibrary', 'storybook']
published: true
published_at: 2023-03-21 23:00
---

UI のテストを AAA パターンで書いていて、ふと「`render()` って Arrange と Act のどっちだっけ？」と疑問に感じました。

この疑問から色々と調査した過程で「UI のテストは AAA パターンより Given-When-Then パターンを意識した方がしっくりくるのではないか」と感じたので、調査結果と一緒に紹介します。

:::message
この記事の大部分は自転車置き場の議論です。
:::

## AAA(Arrange-Act-Assert) パターンとは

AAA パターンとは、テストコードを Arrange（準備）、Act（実行）、Assert（確認）の3つのフェーズに分けて記述する、テストコードを読みやすくするためのパターンのひとつです。それぞれのフェーズで記述する内容は次のようになります。

- Arrange
  - テストを実施するために必要となる前提条件や必要なデータを準備する
    - テスト環境の初期化や依存関係（テストダブルも含む）の構築
    - テスト対象のメソッドに渡すパラメータ
- Act
  - テスト対象の振る舞いを実行する
    - テスト対象のメソッドの実行など
- Assert
  - 期待された結果であるかを確認する

実際に最小限な記述と AAA パターンを見比べてみると、AAA パターンの方が「どのような手順でテストを行っているのか」を表せているように見えます。

```ts
// 最小限
test('reverse 関数は与えられた文字列を反転する', () => {
  expect(reverse('abc')).toBe('cba');
});

// AAA パターン
test('reverse 関数は与えられた文字列を反転する', () => {
  // Arrange
  const input = 'abc';

  // Act
  const result = reverse(input);

  // Assert
  expect(result).toBe('cba');
});
```

UI に関するテストとしても、[`react-dom/test-utils`](https://ja.reactjs.org/docs/test-utils.html) や [Testing Library](https://testing-library.com) でもこの AAA パターンを参考にされています。

React のテストユーティリティである `react-dom/test-utils` では、AAA パターンの Act を由来とする [`act()`](https://ja.reactjs.org/docs/test-utils.html#act) というヘルパーが用意されています。このヘルパーは、レンダリングやユーザーアクション、データの取得などによる UI の更新が、確実に DOM へ反映されることを保証する用途として使用します。また Testing Library においても[最初の例](https://testing-library.com/docs/react-testing-library/example-intro/)が AAA パターンで書かれています。

## UI のテストに AAA パターンを適用した際の疑問

AAA パターンは React でも公式として参考にされていることもあり、良さそうなプラクティスに見えます。ですが実際に UI のテストを AAA パターンで記述していると、判断に迷う点がいくつかありました。

それでは実際に AAA パターンで書かれた UI のテストをみてみましょう。

```tsx:Checkbox.test.tsx
import { fireEvent, render, screen } from '@testing-library/react';
import '@testing-library/jest-dom';
import { Checkbox } from './Checkbox';

test('Checkbox は、非チェック状態の時にクリックすると、チェック状態になる', () => {
  // Arrange
  // 1: render は Arrange か Act か？
  render(<Checkbox />);

  // 2: 「非チェック状態のとき」を確認するのは Assert か？
  expect(screen.getByRole('checkbox')).not.toBeChecked();

  // Act
  fireEvent.click(screen.getByRole('checkbox'));

  // Assert
  expect(screen.getByRole('checkbox')).toBeChecked();
});
```

さて、1の `render()` は Arrange と Act のどちらでしょうか？

この例ではユーザーアクションのクリックが Act となりそうなので Arrange で問題なさそうに思えます。

ですが、ユーザーアクションを必要としないテストだとどうでしょう？Act がないテストで良いのでしょうか？

```tsx
// `render` は Arrange？
test('Checkbox は、初期状態では非チェック状態となる', () => {
  // Arrange
  render(<Checkbox />);

  // Act
  // 実行される振る舞いはない？

  // Assert
  expect(screen.getByRole('checkbox')).not.toBeChecked();
});
```

それとも `render` は Act になりそうでしょうか？

その場合 `props` を渡さないような、準備が不要な UI のテストは Arrange がないテストとなるのでしょうか？
またクリック時のテストでは `render` は Arrange となりそうでしたが、テストによって `render` が Arrange だったり Act だったりしてもいいのでしょうか？

```tsx
// `render` は Act？
test('Checkbox は、初期状態では非チェック状態となる', () => {
  // Arrange
  // 準備はない？

  // Act
  // 他のテストでは `render` は Arrange だったけど？
  render(<Checkbox />);

  // Assert
  expect(screen.getByRole('checkbox')).not.toBeChecked();
});
```

また2の事前条件である「非チェック状態のとき」を確認する Assert についてはどうでしょうか？

これを確認しないと「初めからチェック状態だった場合」にテストがすり抜けてしまうので、確認自体は必要となるでしょう。ですがこの場合、Arrange-Assert-Act-Assert の順となってしまいますが、これでいいのでしょうか？

疑問をまとめると次のようになります。

- 疑問1: `render` は Arrange と Act のどちらになるのか
  - Arrange の場合、ユーザーアクションがない UI のテストは Act がないテストでいいのか
  - Act の場合、`props` を渡さないような準備が不要な UI のテストは Arrange がないテストでいいのか
    - この場合、`render` はテストによって Arrange だったり Act だったりしてもいいのか
- 疑問2: 事前条件の確認はどのフェーズになるのか
  - Assert の場合、Arrange-Assert-Act-Assert の順となっていいのか

子供の頃「刑事コロンボ」が好きだったせいか、細かいことが気になると夜も眠れません。

### 疑問1: `render` は Arrange と Act のどちらになるのか

#### React では何と言っているか

公式情報として React のテストのレシピ集を確認してみると、[レンダリング](https://ja.reactjs.org/docs/testing-recipes.html#rendering)や[イベント](https://ja.reactjs.org/docs/testing-recipes.html#events)は `act` で囲われています。`act` の用途からはレンダリングやユーザーアクション、データの取得といった内容は Act に相当するように見えます。

一方 [Testing Library の例の Arrange](https://testing-library.com/docs/react-testing-library/example-intro/#arrange) では、`render` を Arrange として扱っています。ここで注目したいのは、Testing Library の例には Act にユーザーアクションがあるという点です。つまり、ユーザーアクションがないテストは不明ですが、ユーザーアクションのような UI の変更を伴うテストでは `render` を明確に Arrange として扱っている、ということです。

また `testing-library/react` の `render` は内部で `react-dom/test-utils` の `act` を呼び出していることにも注目した方が良さそうです。

ここまでをまとめると、React の公式情報と Testing Library からは次のことが言えそうです。

- UI の更新を伴うような振る舞い（レンダリングやユーザーアクション、データ取得など）は `act` で囲う
- `act` の由来は AAA パターンの Act である
- UI の更新を伴うような振る舞いを持つテストでは `render` は Arrange としている
- UI の更新を伴うような振る舞いを持たないテストは分からない

#### アサートファーストでの考察

続いてアサートファースト^[テストをアサートから書くテクニック。テスト駆動開発 アサートファースト（p.193）]なアプローチで、AAA パターンを Assert から遡って考察してみました。

まず Assert は、期待した結果であるか確認するフェーズです。UI のテストで確認するべき結果はもちろん表示された結果の UI となるはずです。先ほどの疑問を感じた2つの例から、確認するべき表示された結果の UI を考えると次の2種類になると考えられます。

- 「非チェック状態の時にクリックすると、チェック状態になるテスト」の確認するべき表示結果は、**クリック後の変更が起きた後の表示結果**
- 「初期状態では非チェック状態テスト」の確認するべき表示結果は、**初期状態でのレンダリングした結果**

続いて Act ですが、Assert で確認対象となる結果がヒントとなりました。Act はテスト対象の振る舞いを実行し、その実行された結果を Assert で確認します。つまり言い換えると、**Assert で確認する結果を生み出す振る舞いを実行するフェーズが Act** であると言えそうです。そのため、それぞれのテストの Act は次のようになります。

- 「非チェック状態の時にクリックすると、チェック状態になるテスト」
  - 確認するべき表示結果は、クリック後の変更が起きた後の表示結果
  - その結果を生み出す振る舞い（Act）は、クリック（ユーザーアクション）
- 「初期状態では非チェック状態テスト」
  - 確認するべき表示結果は、初期状態でレンダリングした結果
  - その結果を生み出す振る舞い（Act）は、レンダリング（`render`）

ここまでくると Arrange は簡単です。Arrange はテストを実施するために必要となる前提条件や必要なデータを準備するフェーズなのです。「非チェック状態の時にクリックすると、チェック状態になるテスト」の、クリック後の変更が起きた後の表示結果を生み出す振る舞いに必要な準備は、非チェック状態で表示されていることになります。「初期状態では非チェック状態テスト」の初期状態でレンダリングした結果を生み出す振る舞いに必要な準備はないことになります。

ここまでで判明したことを AAA パターンの各フェーズに補足すると次のようになります。

- Arrange
  - Act で実行する振る舞いに必要な前提条件やデータを準備するフェーズ
- Act
  - Arrange で準備した状況下で、Assert で検証する結果を引き起こす、テスト対象の振る舞いを実行するフェーズ
- Assert
  - Act によって引き起こされた結果が、期待された結果であるかを確認

UI のテストで検証する結果とそれを引き起こすテスト対象の振る舞いをまとめると次のようになります。

| Assert の対象となる検証する結果                                                  | 検証する結果を引き起こす振る舞い（Act）          | Assert 対象の例                                                                            |
| -------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| レンダリングされた結果                                                           | レンダリング（`render`）                         | 初期状態での表示結果や初期表示時にデータ取得をして表示した結果                             |
| 既に表示されている UI に対して、何らかのアクションによって引き起こされた変更結果 | ユーザーアクションなどによって引き起こされた変更 | 既に表示されている結果が、ユーザーアクションや双方向通信などによって変更された後の表示結果 |

つまり論点は「`render` は Arrange と Act のどちらか」ではなく、「Assert で検証対象となる結果を生み出す振る舞い（Act）は何か」を考える必要があったということです。

#### 疑問1に対する答え

疑問1の「`render` は Arrange と Act のどちらになるのか」に対する答えをまとめると次のようになります。

- 「Assert で検証する結果を生み出す振る舞い（Act）は何か」によって `render` は Arrange と Act のどちらにもなりえる
- 検証する結果が、変更を伴わない表示結果の場合
  - Act は「表示結果を生み出す振る舞い（`render`）」
- 検証する結果が、変化が起きた後の表示結果の場合
  - Act は「既に表示されている結果に対して、変化を起こす振る舞い（ユーザーアクションなど）」
  - Arrange は「Act の対象となる、変化が起きる前の表示の準備（`render`）」

何となく納得いく答えになったのではないでしょうか。

ですが白状すると、これを考慮しながら組織でテストを回し続けることは難しいのではと考えています。ここまでの文章量もさることながら、説明するコストが高く、疑問は解消できても結局は迷ってしまいそうと感じています。

### 疑問2: 事前条件の確認はどのフェーズになるのか

疑問2に関しては AAA パターンを命名した Bill Wake 氏の [3A – Arrange, Act, Assert](https://xp123.com/articles/3a-arrange-act-assert/) の記事に回答がありました。要約すると次のようになります。

- そもそも2つのテストに分けることを検討する
- それか、事前条件を確認することが必要ならパターンを崩す価値はある
  - セットアップが複雑で、オブジェクトが期待する初期状態にあることを信頼できない場合など

今回のケースで考えると、「Checkbox は、初期状態では非チェック状態となる」テストがあるので、2 つのテストに分かれている状態になってます。つまり事前条件の「初期状態が非チェック状態か」の確認はなくしていい、と考えられそうです。

反対に「初期状態がどうなっているかは不明」なため確認の必要がある、とも考えられそうです。これは、仕様が「初期状態がチェック状態」に変更された場合や「初期状態が非チェック状態となる」テストが不足している可能性を考慮しています。これはテストケース間に依存関係を持ち込んでしまうため、あまり良くないように感じます。

ここで新たな疑問として、判断基準をどうするか、が生じました。また明確な判断基準を設けることが難しく、人によって判断が分かれる部分でもあります。

#### 疑問2に対する答え

疑問2の答えとしては、必要がある場合は Arrange-Act-Assert の順序を崩してもいい、となります。

ただし、この必要がある場合の判断基準を揃えることは難しいため、疑問が解消されてもテストを書く上での迷いは解消されないのではと思われます。

### なぜこのような疑問が生まれたのか

筆者は AAA パターンを厳密に適用しようとするあまり、テストで「何を検証するのか」の意識が薄まっていたと考えました。

パターンが本来果たしたい目的は、テストの手順を分かりやすく書くことのはずでした。そして分かりやすいテストを書く目的は、テストの質を良くすることです。

パターンを使うことは自転車置き場の議論になりやすく、厳密に適用することばかりを考えてしまっていて本来の目的を見失っていたと実感しています。

もうひとつの要因として、AAA パターンの生まれた背景がクラスベースのプログラムのユニットテストを対象としていたと考えられます。クラスベースのテストでは、ほとんどの場合で Arrange はテスト対象のインスタンスの準備となることが想像しやすいです。コンストラクタのテストなどの例外は考えられますが、メソッドに対するテストの方が圧倒的に多くなるはずです。そうなると Arrange がテスト対象のインスタンスの準備となることが多くなり、シンプルに「Arrange ＝インスタンス化」と捉えてしまっていても差し支えなかったのではないかと推察しました。実際に[テスト駆動開発](https://www.amazon.co.jp/dp/4274217884)では Arrange を「オブジェクトを作る」と言い切っています^[テスト駆動開発 第19章: 前準備（p.151）]。

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->

:::details クラスのテストコードの例

```ts
class Amount {
  public add(addend: Amount): Amount {
    return new Amount(this.value + addend.value);
  }

  public constructor(value: number) {
    if (!Number.isInteger(value)) throw new RangeError();

    this.value = value;
  }

  private readonly value: number;
}

// コンストラクタのテストは例外になるが、メソッドのテストのほうが圧倒的に多くなり、
// 「Arrange＝インスタンス化」と捉えてしまっていても齟齬はなかったのではないか
describe('Amount クラス', () => {
  test('整数以外の値で生成するとエラーを投げる', () => {
    expect(() => {
      new Amount(0.1);
    }).toThrow(RangeError);
  });

  test('add メソッドは、自身の値と渡された値を合算した Amount を返す', () => {
    // Arrange
    const sut = new Amount(100);
    const addend = new Amount(200);

    // Act
    const result = sut.add(addend);

    // Assert
    expect(result).toEqual(new Amount(300));
  });
});
```

:::

<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->

一方 UI のテストでは既に説明したように、`render` は Act と Arrange のどちらにもなりえます。そのため、クラスベースほどシンプルに捉えることができず、結果的に疑問として顕在化したのではと推察しました。

### 残る問題について

さて、疑問は解消しましたが、テストコードを書く上での迷いは残ってしまうと考えられます。

- `render` は Arrange と Act のどちらにもなりえるため、結局は迷いそうな気がする
  - 常に念頭に入れながら実施し続けるのは面倒
  - 説明コストが高すぎる
  - 組織で揃えて実施し続けるのは難しい
- パターンを崩してもいい場合の判断基準に困りそう
  - 明確な判断基準はなく、人によって判断が分かれそう
  - パターンを崩さずに事前条件の確認を別のテストに分ける場合、偽陰性のあるテストスイーツとなる可能性がある
    - 初期状態などの仕様変更によるすり抜け
    - カバレッジ不足によるすり抜け

これら問題は、調査の過程で Given-When-Then パターンを意識すると解消するのではないか、と感じました。

## Given-When-Then パターンとは

Given-When-Then パターンは、ビヘイビア（振る舞い）駆動開発（BDD）の一部として開発された、テストを構造的に表す手法です。

ビヘイビア駆動開発とはテスト駆動開発から発展した、ソフトウェアがどのように振る舞うべきかに焦点を当てた開発手法です。ソフトウェアの振る舞い（機能や動作）、つまり仕様をビジネス上の要件やユーザーストーリーに基づいて定義し、そのストーリーに基づいてテストを作成、記述していきます。そのため仕様の表現がそのままテストコードと一致するようになります。

Given-When-Then パターンはテスト、つまり仕様を次のようなステップで表現します。

- Given
  - 振る舞いを実行する前の状態
  - 「それはどういう前提で？」
- When
  - 発生すると最終結果をもたらす振る舞いやアクション
  - 「それは何をしたら？」
- Then
  - 指定した状態で指定した振る舞いによって生じた想定される結果
  - 「それはどうなってる？」

実はこの Given-When-Then パターンと AAA パターンは内容に変わりはなく、言い方が違うだけです。

| 記述する内容                             | AAA     | Given-When-Then |
| ---------------------------------------- | ------- | --------------- |
| テスト対象の振る舞いを実行するための準備 | Arrange | Given           |
| テスト対象の振る舞いの実行               | Act     | When            |
| 振る舞いによって引き起こされた結果の検証 | Assert  | Then            |

内容に違いはないのですが、Given-When-Then パターンの方が AAA パターンよりもテストを書く上での迷いが少なくなるように感じました。実際に適用したテストコードを見てみましょう。

## UI のテストに Given-When-Then パターンを適用する

先程の AAA パターンのテストを、Given-When-Then パターンで仕様をまとめると次のようになります。

- Checkbox は
- Given:
  - 表示されていて
  - 非チェック状態のときに
- When:
  - クリックすると
- Then:
  - チェック状態となる

この時点で「表示されている」という前提条件が Given に含まれていることから、`render` は Given で行うことが分かります。

同様に「非チェック状態の前提条件」が Given に含まれ、非チェック状態の確認も Given で行うことが分かります。そのため、AAA パターンで起こりえる、パターンを崩すかどうかの判断が不要となります。

それでは同じテストを Given-When-Then パターンで記述してみましょう。

```tsx:Checkbox.test.tsx
import { fireEvent, render, screen } from '@testing-library/react';
import '@testing-library/jest-dom';
import { Checkbox } from './Checkbox';

it('Checkbox は、非チェック状態の時にクリックすると、チェック状態になるべき', () => {
  // Given: 表示されていて
  render(<Checkbox />);
  // Given: 非チェック状態のときに
  expect(screen.getByRole('checkbox')).not.toBeChecked();

  // When: クリックすると
  fireEvent.click(screen.getByRole('checkbox'));

  // Then: チェック状態となる
  expect(screen.getByRole('checkbox')).toBeChecked();
});
```

もうひとつの「Checkbox は、初期状態で表示すると非チェック状態となる」のテストも見てみましょう。

- Checkbox は
- When:
  - 初期状態で表示すると
- Then:
  - 非チェック状態となる

このケースのポイントとしては「初期状態で表示する」ことに対する前提条件、つまり Given に相当するステップはないということが整理できます。

この仕様をテストコードで表すと次のようになります。

```tsx
test('Checkbox は、初期状態で表示すると非チェック状態となる', () => {
  // When: 初期状態で表示すると
  render(<Checkbox />);

  // Then: 非チェック状態となる
  expect(screen.getByRole('checkbox')).not.toBeChecked();
});
```

このように Given-When-Then パターンにすることで、テストの手順を仕様として表現が可能となります。またパターンの適用に意識が向いてしまっても、AAA パターンよりも各ステップで何を記述するべきかが明確になったように見えます。

## 2つのパターンの違い

AAA パターンと Given-When-Then パターンの違いは何でしょうか？AAA パターンから Given-When-Then パターンへ変更した差分を見てみましょう。

```diff tsx
  import { fireEvent, render, screen } from '@testing-library/react';
  import '@testing-library/jest-dom';
  import { Checkbox } from './Checkbox';

- test('Checkbox は、非チェック状態の時にクリックすると、チェック状態になる', () => {
+ it('Checkbox は、非チェック状態の時にクリックすると、チェック状態になるべき', () => {
-   // Arrange
+   // Given
    render(<Checkbox />);
-
    expect(screen.getByRole('checkbox')).not.toBeChecked();

-   // Act
+   // When
    fireEvent.click(screen.getByRole('checkbox'));

-   // Assert
+   // Then
    expect(screen.getByRole('checkbox')).toBeChecked();
  });
```

BDD を意識して `test` から `it` への変更とテストケース名に「〜べき（should）」が追加されています。それ以外は空白行とコメントだけで意味のある差分はまったくと言っていいほどありません。

## Given-When-Then で解消される違和感の正体

AAA パターンと Given-When-Then パターンは違いがないと言っていいほど同じでありながら、なぜ（少なくとも私は）違和感が解消されたように見えるのでしょうか？

筆者は、Given-When-Then パターンのほうが、テストで「何を書くか」ではなく「何をさせるか」のアプローチを取りやすいのではないかと考えています。

Arrange-Act-Assert は、それぞれテストで実施する技術的なニュアンスや内容を含んだ単語に見えます。そのため単語の印象に引きずられ、「ここで何を書くか」を意識してしまうのではないかと考えました。

一方 Given-When-Then は BDD が達成しようとしている「自然言語に近い記述で仕様を表すこと」が可能となります。そのため Arrange-Act-Assert と比べると、各ステップで「何をさせるか」に焦点が当たりやすいのではないかと考えました。

また UI のテストはユーザーアクションなどの振る舞いにって変更された結果をテストすることが多く、BDD ライクなアプローチがマッチしやすいのではと考えています。

## 後日談というか今回のオチ

さてこれでテストコードを書くのに迷わないぞ！

と言いたいところですが、[Storybook のインタラクションテスト](https://storybook.js.org/docs/react/writing-tests/interaction-testing)を併用するともっと簡潔になります。

### Storybook のストーリーをテストで再利用する

まずは先程の Checkbox のインタラクションテストを含めたストーリーを作成します。

```tsx:Checkbox.stories.tsx
import { expect } from '@storybook/jest';
import { ComponentMeta, ComponentStoryObj } from '@storybook/react';
import { screen, userEvent, within } from '@storybook/testing-library';
import { Checkbox } from './Checkbox';

export default {
  component: Checkbox,
} as ComponentMeta<typeof Checkbox>;

export const Default: ComponentStoryObj<typeof Checkbox> = {
  name: '初期状態の時',
};

export const ClickedWithUnchecked: ComponentStoryObj<typeof Checkbox> = {
  name: '非選択状態の時にクリックした時',
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    expect(canvas.getByRole('checkbox')).toBeChecked();

    userEvent.click(canvas.getByRole('checkbox'));
  },
};
```

このストーリーをテストコードで再利用すると次のようになります。

```tsx:Checkbox.test.tsx
import { composeStories } from '@storybook/testing-react';
import { fireEvent, render, screen } from '@testing-library/react';
import '@testing-library/jest-dom';
import * as stories from './Checkbox.stories';

describe('Checkbox', () => {
  const { Default, ClickedWithUnchecked } = composeStories(stories);

  describe('Default（初期状態で表示すると）', () => {
    it('非チェック状態となるべき', () => {
      // When: 初期状態で表示すると
      render(<Default />);

      // Then: 非チェック状態となる
      expect(screen.getByRole('checkbox')).not.toBeChecked();
    });
  });

  describe('ClickedWithUnchecked（非チェック状態の時にクリックすると）', () => {
    it('チェック状態になるべき', async () => {
      // Given: 表示されていて、非チェック状態のときに
      const { container } = render(<ClickedWithUnchecked />);

      // When: クリックすると
      await ClickedWithUnchecked.play({ canvasElement: container });

      // Then: チェック状態になる
      expect(screen.getByRole('checkbox')).toBeChecked();
    });
  });
});
```

注目したい点は、Arrange-Act や Given-When 相当のステップが簡潔になっている点です。振る舞いがないテストでは Act と Given 相当のステップでストーリーをリンダリングするだけです。振る舞いのあるテストでは、Arrange と Given 相当のステップでレンダリングし、Act と When 相当のステップで振る舞いとなるストーリーの `play()` を呼び出すだけになります。

Storybook を使っていない場合と比べてみましょう。

```diff tsx
  // 中略

  // Given
- render(<Checkbox />);
- expect(screen.getByRole('checkbox')).not.toBeChecked();
+ const { container } = render(<ClickedWithUnchecked />);

  // When
- fireEvent.click(screen.getByRole('checkbox'));
+ await ClickedWithUnchecked.play({ canvasElement: container });

  // 中略
```

サンプルコードがシンプルすぎるのであまり大きな差がないように見えますが、実際の開発では Given と When の記述量がもっと多くなるため、効果が高まると感じています。

### Storybook とテストのどちらにどこまで書くか

インタラクションテストでもユーザーアクションやアサーションが書けるため、どちらにどこまで書けばいいのでしょうか？

この迷いも Given-When-Then パターンを意識することで解消されると考えています。具体的には次のようになります。

- インタラクションテストには Given と When
  - Then はテストで実施するためなし
- テストには Then
  - Given と When はストーリーを再利用する

具体的な内容を表でまとめると次のようになります。

|                                      | Given                                          | When                                   | Then                 |
| ------------------------------------ | ---------------------------------------------- | -------------------------------------- | -------------------- |
| 初期表示結果のインタラクションテスト | props やモックの準備など                       | レンダリングの実行                     | なし（テストで実施） |
| 変更結果のインタラクションテスト     | レンダリングや事前条件の検証、モックの準備など | ユーザーアクションなどの振る舞いの実行 | なし（テストで実施） |
| 初期表示結果のテスト                 | なし（ストーリーに委譲）                       | ストーリーのレンダリングの実行         | 実行結果の検証       |
| 変更結果のテスト                     | ストーリーのレンダリング                       | ストーリーの `play` の実行             | 実行結果の検証       |

## まとめ

<!-- textlint-disable ja-technical-writing/ja-no-weak-phrase -->

AAA パターンと Given-When-Then パターンの内容に差はありません。ですが、AAA パターンで書きづらいと感じたことがある方は、Given-When-Then パターンを意識すると迷いなく書けるようになるかもしれません。

<!-- textlint-enable ja-technical-writing/ja-no-weak-phrase -->

## 参考

<!-- markdownlint-disable MD034 -->

http://wiki.c2.com/?ArrangeActAssert=

https://xp123.com/articles/3a-arrange-act-assert/

https://learn.microsoft.com/ja-jp/visualstudio/test/unit-test-basics?view=vs-2019#write-your-tests

https://learn.microsoft.com/ja-jp/dotnet/core/testing/unit-testing-best-practices#arranging-your-tests

https://martinfowler.com/bliki/GivenWhenThen.html

https://cucumber.io/docs/gherkin/reference/#steps

https://qiita.com/hideshis/items/b853f2a0ff4769f24cfb

https://qiita.com/inasync/items/e0b54e62784710c4b42d

https://qreat.tech/4534/

https://lassala.net/2017/07/20/test-style-aaa-or-gwt/

https://zenn.dev/takepepe/articles/storybook-driven-development

https://zenn.dev/azukiazusa/articles/df307292037265

<!-- markdownlint-enable MD034 -->
