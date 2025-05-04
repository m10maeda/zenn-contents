---
title: 'フロントエンドのテスト、もう全部 Storybook でよくない？→よくなかったけどちょうどいい落とし所が見つかった'
emoji: '🧪'
type: 'tech'
topics: ['test', 'storybook', 'vitest', 'msw']
published: false
---

「このフロントエンドのテスト、全部 Storybook でいいんじゃない？」

そんな考えから始まった試行錯誤の記録をご紹介します。

## モチベーション

私が担当しているプロジェクトでは Jest + Storybook + MSW を使用していて、テスト方法としては次のような構成を採用していました。

- Storybook で Arrange と Act までを定義（UI 状態とユーザー操作を Story として表現）
- Jest でその Story をレンダリングし、`play` 関数を実行、結果をアサーション

```tsx
import { composeStories } from '@storybook/testing-react';
import * as stories from './TaskForm.stories';

const { AddTask } = composeStories(stories);

test('タスクを追加できる', async () => {
  const { container } = render(<AddTask />);

  await AddTask.play?.({ canvasElement: container });

  expect(screen.getByText('新しいタスク')).toBeInTheDocument();
});
```

しかし、この方法には以下のような課題がありました。

- Storybook、Jest、MSW の組み合わせによるセットアップの複雑さ
- Storybook と Jest の `body` 要素が一致しないという技術的な問題がある
  - `Portal` を使用した UI コンポーネントでは致命的で、解決に至らず
- テストファイルだけを見ても、Arrange と Act の内容を把握しづらい
- Story とテストコードがファイルとして分離しているため、確認・修正の際に行き来する必要がある
- Jest でのアサーションと Storybook での表示確認が二重作業になってしまう

Storybook のインタラクションテストではアサーションも記述可能です。また、test-runner を使えば CI などでのテスト自動化も実現できます。これらの課題を解決するため、「Storybook のインタラクションテストにアサーションも含め、テストを Storybook に集約できないか」という方向性を検討し始めました。

## 結論：「ユースケースを Story に + Vitest で実行」という実用的な解決策

1. **ユースケースごとに Story をまとめる**

   - ユーザーが実際に体験するコンテキストを再現する
   - ユーザー操作による UI コンポーネントの変化を表現する
   - MSW で API をモックして様々な状態を再現する

2. **Vitest で Story を直接テストとして実行**

   - ブラウザでの視覚的確認とテストの自動化を両立する

3. **E2E テストは別途 Playwright 等で補完**
   - 全体的な流れの確認は E2E テストに任せる
   - 実際のバックエンドとの統合テストは E2E で行う

この方針を元にしたサンプルプロジェクトは以下になります。

<!-- markdownlint-disable MD034 -->

https://github.com/m10maeda/storybook-vitest-integration

<!-- markdownlint-enable MD034 -->

また記事執筆中に Storybook 9 がリリースされました。オプトイン形式ではありますが、Vitest で Story をテストとして実行するアプローチが今後主流になりそうだと感じています。

## Story の書き方：ユーザー視点を重視する

Story を書く際に重要なのは「ユーザーが直面する状況」を再現することです。つまり、ユースケースベースのバリエーションが必要になります。

すべての UI コンポーネントのすべてのバリエーションを網羅する必要はありません。それよりもユーザーの視点で考えることが大切です。

例えば、ToDo アプリの場合を考えてみましょう。

TaskList コンポーネント単体の Story は実は必要ない可能性があります。実際の使用では TaskForm でタスクを追加すると TaskList にも変化が生じます。そのため、TaskList と TaskForm を組み合わせた Story の方が実際のユーザー体験に近くなります。

```tsx
import { expect, userEvent, within } from 'storybook/test';

import { TodoPage as Component } from './todo';

import type { Meta, StoryContext, StoryObj } from '@storybook/nextjs-vite';

const meta = {
  component: Component,
  parameters: {
    layout: 'fullscreen',
  },
} satisfies Meta<typeof Component>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = {
  args: {
    tasks: [
      {
        id: 'c7b3d8e0-5e0b-4b0f-8b3a-3b9f4b3d3b3d',
        title: '卵を買う',
        done: false,
      },
      {
        id: '3074d223-f998-4c38-8beb-1f70c8c3da7b',
        title: '小麦粉を買う',
        done: true,
      },
    ],
  },
};

export const NoTasks: Story = {
  args: {
    tasks: [],
  },
  play: async ({ canvasElement, step }) => {
    const canvas = within(canvasElement);

    await step('"タスクはありません"が表示されていること', () => {
      expect(canvas.getByText('タスクはありません'));
    });
  },
};

export const AddTaskSuccess: Story = {
  ...Default,
  play: async ({ canvasElement, step }) => {
    const canvas = within(canvasElement);

    // 事前条件
    await step('"牛乳を買う"タスクが表示されていないこと', async () => {
      expect(
        canvas.queryByRole('checkbox', { name: '牛乳を買う' }),
      ).not.toBeInTheDocument();
    });

    await step('"牛乳を買う"タスクを追加', async () => {
      const form = canvas.getByRole('form', { name: 'タスク追加' });

      const input = within(form).getByRole('textbox', { name: '新しいタスク' });

      await userEvent.type(input, 'Buy milk');

      const button = within(form).getByRole('button', { name: '追加' });

      await userEvent.click(button);
    });

    // 事後条件
    await step(
      '"牛乳を買う"タスクが未チェック状態で表示されていること',
      async () => {
        const checkbox = canvas.getByRole('checkbox', { name: '牛乳を買う' });

        expect(checkbox).not.toBeChecked();
      },
    );
  },
};
```

## テストの考え方：UI を通したテストを基本とする

基本的に、関数は UI コンポーネントを介してテストすることを推奨します。

UI コンポーネント経由でテストすることが困難な場合のみ、例外として関数を直接テスト対象にするという考え方を採用しています。できるだけ UI コンポーネントを介したテストを心がけることが重要です。

## このアプローチの利点

1. **Story とテストの統合**

   - 同じ Story が UI の視覚的な確認とテストの両方に使える
   - コードの重複が減少し、メンテナンスが容易になる

2. **ユースケースの明確化**

   - 各 Story がユーザーの具体的な操作シナリオに対応
   - プロダクトの動作を理解しやすい形で文書化できる

3. **効率的なテスト実行**

   - Vitest の高速な実行環境でテストを効率的に実行できる
   - CI での自動化も容易

4. **テスティングトロフィーの考え方とマッチ**
   - テスティングトロフィーでは統合テストが最も効率的で、費用対効果に優れているとされている
   - このアプローチは複数のコンポーネントを組み合わせたユースケースベースのテストを重視するため、まさに統合テストに相当する
   - エンドユーザーの視点に近いテストを優先することで、テストの信頼性と価値が高まる

## 他のアプローチとの比較

### 方法1：Storybook の test-runner を使用する方法

当初は「Storybook の test-runner を使用して、すべてのテストを Storybook に集約する」という案を検討していましたが、以下の課題があります。

- UI コンポーネントを介さない関数単体でのテストができない
- Storybook の立ち上げやビルド後の成果物が必要になるため非効率的

### 方法2：Jest や Vitest で Story を読み込んでテストする方法

`composeStories` を使用して Jest や Vitest などのテスト内で `play` 関数を呼び出す方法です。この方法では以下の課題があります。

- MSW と組み合わせるとセットアップが複雑になる（特に Jest の場合）
- Portal の対応が技術的に難しい
- テストと Story が分離することで可読性が低下する
- 単純なレンダリング結果に対する検証が冗長になる
  - props のバリエーションによる違いなどで、「◯◯が表示されていること」を検証する際に、Storybook 上とテストの両方で二重に確認している状態になる

## まとめ

「フロントエンドのテストは全部 Storybook でよくない？」という問いに対する結論は、「完全に Storybook だけでは難しい」ということです。ただし、適切な落とし所となるテストの方針と具体的な方法が見つかりました。

- ユーザー視点の Story を作成する
- Story にインタラクションとアサーションを含める
- Vitest で Story を直接テストとして実行する

この方法であれば、可視性・再利用性・開発体験のバランスが取れた、実用的なテスト戦略が実現できると考えています。
