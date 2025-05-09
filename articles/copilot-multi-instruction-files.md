---
title: 'GitHub Copilot の指示書が複数ファイル対応に！ルールを用途別に整理できる新機能'
emoji: '📚'
type: 'tech'
topics: ['githubcopilot', 'vscode', 'ai']
published: true
---

## GitHub Copilot の指示がもっと柔軟に！VSCode v1.100.0 の新機能

GitHub Copilot にカスタム指示を与える `.github/copilot-instructions.md`。  
これまで、この指示書は**1ファイルのみ対応**で、すべてのルールやガイドラインを1つに詰め込む必要がありました。

そのため、以下のような課題がありました。

- 内容が肥大化して読みにくい
- 技術スタックごとのルールを分けづらい
- チームでの管理・メンテが大変

## 複数ファイルで指示を管理できるように

[VSCode v1.100 のリリース](https://code.visualstudio.com/updates/v1_100)により、`.github/instructions/*.instructions.md` にファイルを分割して配置できるようになりました。  
これにより、Copilot のカスタマイズがより柔軟になりました！

`.clinerules` など一部のツールでは以前から使われていた構成法ですが、Copilot にもようやく導入された形です。

[詳しい設定方法や仕様は公式ドキュメント](https://code.visualstudio.com/docs/copilot/copilot-customization#_use-instructionsmd-files)をご参照ください。

## ディレクトリ構成例

以下のように用途別にファイルを作成できます。

```plain
.github/
  instructions/
    general.instructions.md
    react.instructions.md
    testing.instructions.md
```

## `.instructions.md` ファイルの書き方

基本的な書式は以下の通りです。

```md
---
applyTo: '**'
---

GitHub Copilot に適用したい指示をここに記載します。
```

### applyTo の役割

- applyTo は Front Matter で、この指示をどのファイルに適用するかを指定できる
- glob パターンが使える
- applyTo を省略すると、すべてのファイルに適用される

## 具体的な例

### 全般的なコーティングルール

```md:.github/instructions/general.instructions.md
---
applyTo: "**"
---

- 変数名や関数名は意味のある名前を使ってください。
- コメントは簡潔かつ具体的に書いてください。
- マジックナンバーは避け、定数として定義してください。
- コードの可読性を重視してください。
```

### React コンポーネント

```md:.github/instructions/react.instructions.md
---
applyTo: "src/components/**/*.tsx"
---

- クラスコンポーネントではなく、関数コンポーネントとフックを使ってください。
- props は TypeScript の interface または type で定義してください。
- スタイリングには styled-components を使用してください。
- JSX 内ではロジックを複雑にしすぎず、必要であれば関数に切り出してください。
```

### テスト

```md:.github/instructions/react.instructions.md
---
applyTo: "src/**/*.test.tsx,src/**/*.spec.tsx,src/**/*.test.ts,src/**/*.spec.ts"
---

- テストは `@testing-library/react` を使って記述してください。
- 外部モジュールは `vi.mock` を使ってモック化してください。
- 実装ではなく、ユーザーの操作や見た目の振る舞いをテストしてください。
- describe / it ブロックの説明は何をテストしているか明確に記述してください。
```

## まとめ

- VSCode v1.100.0 以降、Copilot の指示書が複数ファイルで管理できるようになった！
- 用途別・対象ファイル別に指示を分けられることで、柔軟性・可読性・メンテナンス性が大幅向上！
- チーム開発や大型プロジェクトでも、Copilot の精度がさらに引き出せる！

AI アシスタントを活かすには「コンテキスト」が命。  
この機能を使えば、Copilot に**より明確で的確なガイドライン**を渡せるようになります。  
ぜひ活用して、**あなたの Copilot を真の相棒に育てていきましょう！**
