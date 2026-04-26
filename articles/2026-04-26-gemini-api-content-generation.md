---
title: "Gemini APIで日本語コンテンツを量産するシステムの設計思想とプロンプトパターン"
emoji: "✨"
type: "idea"
topics: ["TypeScript", "Next.js", "AI", "自動化", "Vercel"]
published: true
---

## はじめに

Futuristic Imagination LLC代表エンジニアの佐藤琢也です。

いきなりですが、皆さん、日本語コンテンツの生成、AIでどうやってますか？

うちでは、現在11サイトのAIオウンドメディアを、Next.js + Gemini API + Vercel Cronを使って僕一人で自動運用しています。毎日せっせと記事を自動生成して、SNSにも自動投稿。これが意外と回るんです。

正直、最初は「本当に使えるのか？」「日本語は大丈夫か？」と半信半疑でした。僕自身、いくつかのプロジェクトでWordPressからNext.jsへの移行や、SNS自動化、そして何よりGemini APIを使ったコンテンツ生成パイプラインの構築に携わってきました。その中で培った実体験と、ぶっちゃけ「失敗も多かったな」という反省点も含めて、今回は僕が実際に運用しているシステムの設計思想と、特に重要なプロンプトパターンについて深掘りしていきたいと思います。

この記事は、

*   Gemini APIで日本語コンテンツを量産したい
*   Next.jsとVercel Cronを組み合わせて自動化したい
*   プロンプトエンジニアリングに悩んでいる
*   一人で複数のメディアを運用したいけど手が回らない

といった方に読んでいただけると嬉しいです。僕の実体験ベースなので、泥臭い部分も正直にお話しします。

## 11サイトを1人で運用するためのシステム全体像

まず、僕が11サイトのAIオウンドメディアを一人で回しているシステムの全体像からお話ししましょう。

:::details アーキテクチャ図（簡易版）
```mermaid
graph TD
    A[Vercel Cron (スケジュール実行)] --> B{Next.js API Route}
    B --> C[Gemini API (コンテンツ生成)]
    C --> D[コンテンツ保存 (例: Cloud Storage)]
    D --> E[Next.js (SSG/ISR)]
    E --> F[Webサイト公開]
    B --> G[SNS投稿API]
    G --> H[SNSアカウント]
```
:::

主な流れはこんな感じです。

1.  **Vercel Cron**が指定された時間に**Next.jsのAPI Route**を呼び出す。
2.  API Route内で、**Gemini API**を呼び出し、記事コンテンツを生成。
3.  生成されたコンテンツをRDBやNoSQLなど（僕の場合はFirestoreをよく使います）に保存。
4.  Next.jsのアプリケーションが、保存されたコンテンツを元に記事ページを生成し公開（SSGやISRを活用）。
5.  必要に応じて、生成された記事の情報を元に**SNS投稿API**（XのAPIなど）を呼び出し、自動でSNSに投稿。

このシンプルなパイプラインをサイトごとに構築しています。ポイントは、各サイトで異なるテーマやペルソナを設定し、それに応じたプロンプトを適用している点です。

### なぜNext.jsとVercel Cronなのか

「なぜWordPressじゃないの？」と聞かれることも多いんですが、僕がNext.jsを選んだ理由はいくつかあります。

*   **開発体験の良さ**: TypeScriptとの相性が抜群で、型安全な開発がしやすい。
*   **パフォーマンス**: SSG/ISRによる高速なページ表示。SEOにも有利です。
*   **API Route**: バックエンド処理をNext.js内で完結できる。Vercel Functionsとしてデプロイされるため、サーバー管理の手間が少ない。
*   **Vercel Cron**: 無料枠でもかなり使える。スケジュール実行の設定がVercelのダッシュボードで完結し、コードベースで管理できるのが最高です。

正直、WordPressも良い選択肢ですが、カスタム性が求められるAI連携や、高速なサイト表示、そして何より開発者目線での管理のしやすさを考えると、Next.js + Vercel Cronは今の僕にはベストな組み合わせでした。

## 日本語コンテンツ生成のためのプロンプト設計思想

さて、本題のプロンプト設計です。Gemini APIで日本語コンテンツを量産する上で、一番苦労し、一番時間をかけたのがプロンプトの設計でした。

僕のプロンプト設計思想の根幹は、「**AIを優秀なライターとして指示する**」ではなく、「**AIを優秀なアシスタントとして、具体的なタスクと制約を明確に与える**」というものです。

### 失敗談：最初期のプロンプトと「謎の日本語」

運用初期、僕のプロンプトは非常にシンプルでした。

```
あなたはSEOライターです。キーワード「〇〇」について、ブログ記事を執筆してください。
```

これで生成される記事は、一見すると「それっぽい」日本語でした。しかし、よく読むと「ん？」となるような、論理の飛躍や、不自然な言い回し、さらには「この情報、どこから持ってきた？」という幻覚（Hallucination）も頻繁に発生しました。

特に日本語のニュアンスを理解しきれていないような箇所が目立ち、これをそのまま公開するのはリスクが高いと判断しました。SEO的にもユーザー体験的にも、これではダメだ、と。

この失敗から学んだのは、AIに「お任せ」は危険だということ。特に日本語の細かなニュアンスや文化的な背景をAIに完璧に理解させるのは至難の業です。そこで、以下のようなプロンプトパターンにたどり着きました。

### プロンプトパターン1：明確な役割、ターゲット、トーン＆マナー

まず基本中の基本ですが、AIに「誰になって」「誰に向けて」「どんな口調で」書くかを明確に指示します。

```typescript:src/utils/prompts.ts
export const createRolePrompt = (topic: string) => `
あなたはプロのコンテンツライターです。
以下の制約条件と指示に従って、最高品質の日本語記事を作成してください。

# 記事のテーマ
${topic}

# 読者ターゲット
このテーマに興味を持つ初心者や、情報収集をしている一般の方。専門知識がない人でも理解できるよう、平易な言葉遣いを心がけてください。

# トーン＆マナー
フレンドリーで親しみやすく、かつ信頼性のある情報を提供するようにしてください。専門用語を使う場合は、必ず簡単な解説を加えてください。

# 記事の目的
読者に〇〇の基本を理解させ、次の行動（例: 〇〇を試す、〇〇についてさらに調べる）を促す。

# 出力形式
Markdown形式で出力してください。見出し構造（# ## ###）を適切に使用し、箇条書きや太字なども効果的に活用してください。
`;
```

これだけでも品質は格段に上がりますが、これだけではまだ不十分です。

### プロンプトパターン2：ステップバイステップの指示

人間が複雑なタスクをこなす際に、漠然とした指示より、具体的なステップを踏んだ指示の方が良い結果を出すのと同様に、AIにもステップバイステップで指示を出します。

僕の場合、記事生成を以下のステップに分割しています。

1.  **記事タイトルの提案**
2.  **構成案（目次）の作成**
3.  **各セクション本文の執筆**
4.  **導入文、まとめの執筆**

これをGemini APIの複数回の呼び出し（またはFunction Callingなど）で実現するか、一回の呼び出しでまとめて生成させるか、はケースバイケースです。個人的には、**品質を重視するならステップ分割、速度とコストを重視するなら一括生成**を検討します。今はステップ分割を採用しています。

#### 例：構成案作成のプロンプト

```typescript:src/services/gemini.ts
import { GoogleGenerativeAI } from '@google/generative-ai';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function generateArticleOutline(topic: string, articleTitle: string): Promise<string[]> {
  const model = genAI.getGenerativeModel({ model: "gemini-pro" });

  const prompt = `
  あなたはプロのコンテンツライターです。
  以下のテーマと記事タイトルに基づき、読者に価値を提供するブログ記事の構成案（目次）を作成してください。

  # 記事のテーマ
  ${topic}

  # 記事タイトル
  ${articleTitle}

  # 構成案の要件
  *   導入とまとめを含め、5〜7個の大見出し（##）を作成してください。
  *   各大見出しの下に、2〜3個の中見出し（###）を含めてください。
  *   読者が記事全体の内容を把握できるよう、見出しは具体的かつ魅力的にしてください。
  *   出力はMarkdownの目次形式で、見出しのみをリストアップしてください。本文は不要です。

  # 出力例
  ## はじめに
  ### 〇〇とは？
  ### なぜ今〇〇が注目されるのか

  ## 〇〇の主なメリット
  ### メリット1：〇〇
  ### メリット2：〇〇

  ...
  `;

  const result = await model.generateContent(prompt);
  const response = result.response;
  const text = response.text();

  // ここでMarkdownから見出しをパースするロジックが必要
  // 簡単な例として、行ごとに分割し、'##' '###'を含む行を抽出
  return text.split('\n').filter(line => line.startsWith('## ') || line.startsWith('### '));
}
```

このように、具体的な出力形式まで指示することで、AIは迷わず目的に沿った出力をしてくれます。

### プロンプトパターン3：参照情報と制約の付与

AIの「幻覚」を防ぎ、信頼性の高い情報を生成させるために、**参照情報**を与えることは非常に重要です。また、「〇〇してはいけない」「〇〇を含めるな」といった**制約**も効果的です。

```typescript:src/services/gemini.ts
// ... (前略)

export async function generateSectionContent(
  topic: string,
  articleTitle: string,
  sectionTitle: string,
  contextualInfo: string // 参照情報として前のセクションの内容やキーワードリストなどを渡す
): Promise<string> {
  const model = genAI.getGenerativeModel({ model: "gemini-pro" });

  const prompt = `
  あなたはプロのコンテンツライターです。
  以下の記事のテーマ、タイトル、および文脈情報を考慮し、指定されたセクションについて最高品質の日本語で執筆してください。

  # 記事のテーマ
  ${topic}

  # 記事タイトル
  ${articleTitle}

  # 現在執筆中のセクション
  ${sectionTitle}

  # 文脈情報・参照情報
  これまでの記事内容や、このセクションで言及すべきキーワードなどが含まれます。
  ${contextualInfo || "特にありません。"}

  # 執筆要件
  *   日本語で自然かつ読みやすい文章を作成してください。
  *   読者ターゲット（初心者向け）を意識し、専門用語は避け、分かりやすい言葉で説明してください。
  *   このセクションの内容にのみ焦点を当て、他のセクションの内容を繰り返し言及しないでください。
  *   誤情報や不確かな情報は絶対に含めないでください。
  *   過度な断定的な表現は避け、必要に応じて「～と考えられます」「～が一般的です」といった表現を使用してください。
  *   文字数は〇〇字程度を目安にしてください。（具体的な数値を入れる）
  *   見出し（# ## ###）は既にあるので、本文のみを記述してください。

  # 出力形式
  Markdown形式で、指定されたセクションの本文のみを記述してください。
  `;

  const result = await model.generateContent(prompt);
  const response = result.response;
  return response.text();
}
```

`contextualInfo`には、例えば記事全体のキーワードリストや、直前のセクションで書かれた内容の要約などを渡しています。これにより、記事全体の一貫性を保ちつつ、特定のセクションに集中した執筆を促すことができます。

また、「誤情報や不確かな情報は絶対に含めないでください」といったネガティブプロンプトは、幻覚のリスクを減らす上で非常に有効です。

### プロンプトパターン4：few-shotプロンプティング

Gemini APIは、例示（few-shot）によって出力の品質を高めることができます。特定のトーンやスタイルの記事、あるいは特殊なフォーマットを求める場合に効果的です。

これはプロンプト例が長くなるためコード例は割愛しますが、プロンプトの冒頭に「以下は良い記事の例です」として、期待する出力に近い記事例をいくつか提示します。

```
あなたはプロのコンテンツライターです。
以下の例を参考に、記事を執筆してください。

# 良い記事の例1
テーマ: [例1のテーマ]
タイトル: [例1のタイトル]
本文:
[例1の本文]

# 良い記事の例2
テーマ: [例2のテーマ]
タイトル: [例2のタイトル]
本文:
[例2の本文]

# 今回執筆する記事
テーマ: [今回のテーマ]
タイトル: [今回のタイトル]
本文:
```

これは特に、ブランド固有のボイス＆トーンをAIに学習させたい場合に非常に強力です。

## Vercel Cron と Next.js API Route での自動化

プロンプトが固まったら、いよいよ自動化です。僕のシステムでは、Vercel CronをトリガーとしてNext.jsのAPI Routeを実行しています。

### Vercel Cronの設定

Vercel Cronの設定は `vercel.json` に記述します。

```json:vercel.json
{
  "crons": [
    {
      "path": "/api/generate-article",
      "schedule": "0 0 * * *" // 毎日0時0分に実行
    }
  ]
}
```

この例では毎日0時0分に `/api/generate-article` というAPI Routeが呼び出されます。複数のサイトがある場合は、サイトごとに異なるAPI Routeや、共通のAPI RouteにサイトIDなどをパラメータとして渡すことで対応します。

### Next.js API Routeの実装

API Routeでは、先ほど作成したGemini API呼び出しのロジックを実行します。

```typescript:pages/api/generate-article.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { generateArticleOutline, generateSectionContent } from '../../src/services/gemini';
import { saveArticleToDatabase } from '../../src/services/database'; // 仮のDB保存関数

type Data = {
  message: string;
  articleId?: string;
  error?: string;
};

export default async function handler(req: NextApiRequest, res: NextApiResponse<Data>) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method Not Allowed' });
  }

  try {
    const topic = "次世代のAI技術とそのビジネスへの影響"; // 例として固定。実際はDBから取得するなど
    const articleTitle = "【徹底解説】AIが変える未来のビジネス戦略：次世代技術の最前線"; // 仮のタイトル

    // 1. 記事構成案の生成
    const outline = await generateArticleOutline(topic, articleTitle);
    if (!outline || outline.length === 0) {
      throw new Error('記事構成案の生成に失敗しました。');
    }

    let fullArticleContent = `# ${articleTitle}\n\n`; // 記事タイトルを先頭に追加
    let previousSectionContent = ''; // 文脈情報として前のセクションの内容を保持

    // 2. 各セクションの本文生成
    for (const sectionTitle of outline) {
      fullArticleContent += `${sectionTitle}\n\n`; // 見出しを追加
      const sectionContent = await generateSectionContent(topic, articleTitle, sectionTitle, previousSectionContent);
      fullArticleContent += `${sectionContent}\n\n`;
      previousSectionContent = sectionContent; // 次のセクションのために現在の内容を保持
    }

    // 3. データベースに保存
    const articleId = await saveArticleToDatabase({
      title: articleTitle,
      content: fullArticleContent,
      status: 'draft', // 最初は下書きで保存
      generatedAt: new Date(),
    });

    res.status(200).json({ message: '記事の自動生成が完了しました。', articleId });

  } catch (error: any) {
    console.error('記事生成エラー:', error);
    res.status(500).json({ message: '記事生成中にエラーが発生しました。', error: error.message });
  }
}
```

この `generate-article` API RouteがVercel Cronによって定期的に実行され、新たな記事が自動生成されてデータベースに保存されます。あとはNext.jsのビルドプロセスで、これらの記事を公開サイトに反映させるだけです。

僕の運用では、生成された記事はすぐに公開するのではなく、一度「下書き」として保存し、**簡単な自動レビュー**（キーワード出現率チェック、文字数チェック、不適切表現チェックなど）を挟んでいます。最終的には、品質担保のために**人力での最終レビュー**を通すこともあります。完全自動化は魅力的ですが、特に日本語コンテンツではまだまだ人間の目が必要だと感じています。

## 課題と今後の展望

ここまで僕のシステムの設計思想と実装の一部をご紹介してきましたが、もちろん課題も山積です。

*   **Hallucination（幻覚）問題**: プロンプトでかなり抑制できますが、ゼロにはなりません。特に専門性の高い分野では、参照情報の精度が重要になってきます。RAG (Retrieval-Augmented Generation) の導入は必須だと感じています。
*   **コンテンツの多様性**: 同じプロンプトだと似たようなトーンや構造の記事ばかりになりがちです。プロンプトのバリエーションを増やしたり、AIに異なるペルソナを持たせたりする工夫が必要です。
*   **SEO効果の検証**: 自動生成記事が実際にSEOでどれだけ効果があるのか、継続的なデータ分析と改善が必要です。単なる記事量産だけでなく、ユーザーが本当に価値を感じる記事になっているか、常に問い続ける必要があります。
*   **画像生成の統合**: 記事にマッチした画像を自動生成・挿入する機能は、ユーザー体験を向上させる上で不可欠です。Gemini APIのVisionモデルや、DALL-E3、Stable Diffusionなどとの連携を模索中です。

これらの課題に一つずつ向き合いながら、より高品質で、より効率的なコンテンツ生成システムへと進化させていきたいと考えています。

## まとめ

本記事では、Futuristic Imagination LLC代表エンジニアの佐藤琢也として、僕が実際に11サイトのAIオウンドメディアを運用している経験から、Gemini APIを使った日本語コンテンツ量産システムの設計思想とプロンプトパターンについて解説しました。

重要なポイントは以下の通りです。

*   **明確な役割、ターゲット、トーン＆マナーの指示**
*   **ステップバイステップでのタスク分割**
*   **参照情報と明確な制約の付与**
*   **few-shotプロンプティングによる品質向上**

そして、Next.jsとVercel Cronを組み合わせることで、これらのプロンプトを元にしたコンテンツ生成を効率的に自動化できることを示しました。

AIによるコンテンツ生成は、まだまだ発展途上の技術です。しかし、適切なプロンプトエンジニアリングとシステム設計によって、一人でも複数のメディアを効率的に運用できるレベルにまで来ています。

僕自身、たくさんの失敗を繰り返しながら現在のシステムを作り上げてきました。特に日本語のニュアンスや文化的な背景をAIに理解させることの難しさを痛感しています。これからも、より高品質で、よりユーザーに価値を届けるコンテンツを生成できるよう、日々研究と改善を続けていきます。

今回紹介したようなAI自動化システムの構築代行も承っています。コンテンツ制作の効率化、Next.jsへの移行、SNS自動化などでお困りの方は、ぜひFuturistic Imagination LLCにご相談ください。→ https://www.futuristicimagination.co.jp/service/

## 参考

*   Google Gemini API: [https://ai.google.dev/](https://ai.google.dev/)
*   Next.js ドキュメント: [https://nextjs.org/docs](https://nextjs.org/docs)
*   Vercel Cron ドキュメント: [https://vercel.com/docs/concepts/solutions/cron-jobs](https://vercel.com/docs/concepts/solutions/cron-jobs)