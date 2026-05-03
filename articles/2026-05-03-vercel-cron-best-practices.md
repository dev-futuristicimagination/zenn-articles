---
title: "Vercel Cronの本番運用で学んだベストプラクティス：タイムアウト・エラー・並列化"
emoji: "⚡"
type: "tech"
topics: ["TypeScript", "Next.js", "AI", "自動化", "Vercel"]
published: true
---

## はじめに

Futuristic Imagination LLC 代表の佐藤琢也です。弊社ではAIオウンドメディアを11サイト運営しており、Next.jsとGemini API、そしてVercel Cronを組み合わせることで、**毎日数千〜数万文字の記事を自動生成し、パブリッシュする仕組み**を構築しています。まさに「補給不要の自販機型SaaS」を目指し、一人で大きな価値を生み出す完全自動化を日々追求しています。

この運用の中で、Vercel Cronは非常に重要な役割を担っています。しかし、本番環境での安定稼働を実現するまでには、タイムアウトや予期せぬエラー、複数の並列処理の管理など、様々な課題に直面してきました。

本記事では、私が実際に経験し、解決してきたVercel Cronの本番運用におけるベストプラクティスを、具体的なコード例を交えながらご紹介します。Vercel Cronをこれから導入する方、あるいはすでに導入していて安定稼働に課題を感じている方の参考になれば幸いです。

## Vercel Cronとは何か、なぜ重要なのか

Vercel Cronは、Vercelのプロジェクト内で定期的にコードを実行できる機能です。Next.js API RoutesやEdge Functionsとしてデプロイしたエンドポイントを、指定した間隔（毎日、毎週、カスタムスケジュールなど）で自動的に呼び出すことができます。

弊社のようなコンテンツ自動生成システムでは、以下のようなタスクをVercel Cronで実行しています。

*   **記事の自動生成と保存:** Gemini APIを呼び出し、記事コンテンツを生成し、データベースやファイルストレージに保存します。
*   **Webサイトへのデプロイ:** 保存された記事コンテンツを元に、Next.jsのSSG（Static Site Generation）ビルドをトリガーし、サイトを更新します。
*   **SNSへの自動投稿:** 生成された記事のURLやサマリーを元に、X (旧Twitter)、LinkedIn、Note、Qiita、Zennなどの各種SNSに自動投稿します。
*   **SEOパフォーマンスの監視と改善:** Google Search Console APIやGA4 APIと連携し、低パフォーマンス記事の検出、リライト候補の抽出、再デプロイなどを行います。

これらのタスクを自動化することで、人的リソースを最小限に抑えつつ、コンテンツの鮮度を保ち、SEO効果を最大化することができます。

## Vercel Cron 本番運用で直面した課題と解決策

### 課題1: Cronジョブのタイムアウト

Vercelの無料プランでは、サーバーレス関数の実行時間には上限があります。デフォルトでは10秒、Proプラン以上でも最大60秒です。記事生成のようにGemini APIとの複数回やり取りや、画像を生成するような重い処理では、このタイムアウトに引っかかることが頻繁にありました。

**解決策: 処理の分割とキューイング**

一つのCronジョブで全ての処理を完結させようとせず、処理を細かく分割し、それぞれを独立したジョブとして実行するか、キューシステムを導入しました。

例えば、記事生成プロセスを以下のように分割します。

1.  **記事テーマの選定ジョブ:** 記事のキーワードやテーマを決定し、次のジョブに渡すための情報を保存する。
2.  **記事コンテンツ生成ジョブ:** 選定されたテーマに基づき、Gemini APIを呼び出して記事の本文を生成する。この際、本文が長い場合はセクションごとに分割して生成し、途中で保存する。
3.  **画像生成・最適化ジョブ:** 記事に必要な画像を生成し、最適化する。
4.  **記事公開・デプロイジョブ:** 生成された記事と画像を組み合わせて最終的な記事として保存し、Vercelのデプロイをトリガーする。

これにより、各ジョブの実行時間を短縮し、タイムアウトのリスクを大幅に削減できます。弊社では、さらに踏み込んで**Redisなどのキューシステム**を導入し、ジョブ間でデータを渡しながら非同期に処理を進めるアーキテクチャを採用しています。

```typescript:pages/api/cron/generate-article-queue.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { Redis } from '@upstash/redis'; // 例: Upstash Redisを使用

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method Not Allowed' });
  }

  // Vercel Cronからの認証
  const authHeader = req.headers['authorization'];
  if (!authHeader || authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return res.status(401).json({ message: 'Unauthorized' });
  }

  try {
    // 記事テーマの選定処理 (簡易版)
    const themes = ['Next.jsの最新機能', 'AI開発の未来', 'Vercel Cron活用術'];
    const selectedTheme = themes[Math.floor(Math.random() * themes.length)];

    // キューに記事生成タスクを追加
    await redis.lpush('article_generation_queue', JSON.stringify({ theme: selectedTheme, jobId: Date.now() }));

    console.log(`Added article generation task for theme: "${selectedTheme}" to queue.`);
    return res.status(200).json({ message: 'Article generation task queued successfully' });
  } catch (error) {
    console.error('Error queuing article generation task:', error);
    return res.status(500).json({ message: 'Failed to queue article generation task' });
  }
}
```
そして、別のCronジョブが定期的にこのキューを監視し、タスクを処理します。

```typescript:pages/api/cron/process-article-queue.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { Redis } from '@upstash/redis';
import { generateArticleContent } from '../../lib/articleGenerator'; // Gemini APIを呼び出す関数を想定

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method Not Allowed' });
  }

  const authHeader = req.headers['authorization'];
  if (!authHeader || authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return res.status(401).json({ message: 'Unauthorized' });
  }

  try {
    // キューからタスクを一つ取得
    const taskString = await redis.rpop('article_generation_queue');

    if (!taskString) {
      console.log('No article generation tasks in queue.');
      return res.status(200).json({ message: 'No tasks to process' });
    }

    const task = JSON.parse(taskString) as { theme: string; jobId: number };
    console.log(`Processing article generation task for theme: "${task.theme}" (Job ID: ${task.jobId})`);

    // 記事コンテンツの生成 (Gemini API呼び出しなど、重い処理)
    const articleContent = await generateArticleContent(task.theme); // generateArticleContentは外部で定義

    // 生成されたコンテンツをDBに保存するなど、次のステップへ
    // 例: await saveArticleToDB(task.theme, articleContent);

    // 必要に応じて次のキューにタスクを追加することも可能
    // await redis.lpush('article_publishing_queue', JSON.stringify({ ...task, content: articleContent }));

    console.log(`Successfully processed article for theme: "${task.theme}"`);
    return res.status(200).json({ message: 'Article generation task processed successfully' });
  } catch (error) {
    console.error('Error processing article generation task:', error);
    return res.status(500).json({ message: 'Failed to process article generation task' });
  }
}
```

### 課題2: Cronジョブの並列実行とリソース競合

複数のメディアサイトを運用している場合、それぞれのサイトで記事生成やSNS投稿を行うCronジョブが同時に実行される可能性があります。これにより、APIレートリミットに達したり、共有リソース（データベースなど）で競合が発生したりする問題がありました。

**解決策: ロック機構とセッション管理**

並列実行による競合を防ぐために、**分散ロック機構**を導入しました。具体的には、Redisの`SETNX`（SET if Not eXists）コマンドを利用して、ジョブの実行中にロックを取得し、完了後に解放する仕組みです。

```typescript:lib/jobLock.ts
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// ロックを取得する関数
export async function acquireLock(lockKey: string, expirySeconds: number = 60): Promise<boolean> {
  // SETNXでロックを取得。成功すれば1、失敗すれば0が返る。
  // EXオプションで有効期限を設定し、デッドロックを防ぐ
  const result = await redis.set(lockKey, 'locked', { nx: true, ex: expirySeconds });
  return result === 'OK';
}

// ロックを解放する関数
export async function releaseLock(lockKey: string): Promise<void> {
  await redis.del(lockKey);
}
```

このロック機構をCronジョブ内で利用します。

```typescript:pages/api/cron/safe-article-generation.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { acquireLock, releaseLock } from '../../lib/jobLock';
import { generateArticleContent } from '../../lib/articleGenerator';

const LOCK_KEY = 'article_generation_lock'; // このジョブ固有のロックキー

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method Not Allowed' });
  }

  const authHeader = req.headers['authorization'];
  if (!authHeader || authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return res.status(401).json({ message: 'Unauthorized' });
  }

  let lockAcquired = false;
  try {
    // ロックの取得を試みる (有効期限は60秒)
    lockAcquired = await acquireLock(LOCK_KEY, 120); // 処理が120秒を超えないように設定

    if (!lockAcquired) {
      console.log('Another instance of article generation is already running. Skipping.');
      return res.status(200).json({ message: 'Another instance is running, skipped.' });
    }

    console.log('Lock acquired. Starting article generation...');
    const theme = 'AIとWeb開発の最新トレンド'; // 例
    const articleContent = await generateArticleContent(theme);
    // ... 記事の保存やデプロイ処理 ...
    console.log('Article generation completed.');

    return res.status(200).json({ message: 'Article generation process completed successfully.' });

  } catch (error) {
    console.error('Error in safe article generation:', error);
    return res.status(500).json({ message: 'Failed to complete article generation.' });
  } finally {
    if (lockAcquired) {
      await releaseLock(LOCK_KEY);
      console.log('Lock released.');
    }
  }
}
```

このアプローチにより、複数のCronジョブが同時にトリガーされた場合でも、実際に処理を実行するのはロックを取得できたジョブのみとなり、競合を効果的に防ぐことができます。

また、Gemini APIのような外部APIについては、**弊社独自のAI Session Manager (AG Hub)** を構築し、複数エージェントを同時並行で動かすためのセッション管理やレートリミット管理を行っています。これにより、APIの利用状況を集中管理し、効率的な利用を可能にしています。

### 課題3: エラーハンドリングと通知の欠如

Cronジョブはバックグラウンドで実行されるため、エラーが発生しても気づきにくいという問題があります。特に、APIの認証エラーや外部サービスの一時的な障害などでジョブが失敗した場合、放置すると長期的にサイト運用に影響が出ます。

**解決策: 厳格なエラーハンドリングとDiscord通知**

全てのCronジョブで`try-catch`ブロックを徹底し、エラーが発生した際には詳細なログを出力するとともに、**Discordなどのチャットツールに通知を飛ばす仕組み**を構築しました。

```typescript:lib/notification.ts
import axios from 'axios';

export async function sendDiscordNotification(message: string): Promise<void> {
  const webhookUrl = process.env.DISCORD_WEBHOOK_URL;
  if (!webhookUrl) {
    console.warn('DISCORD_WEBHOOK_URL is not set. Skipping Discord notification.');
    return;
  }

  try {
    await axios.post(webhookUrl, {
      content: message,
      username: 'Vercel Cron Notifier', // ボットの名前
      avatar_url: 'https://assets.vercel.com/image/upload/v1641049925/front/vercel/doodle/Cron-Jobs-doodle.svg' // ボットのアイコン
    });
  } catch (error) {
    console.error('Failed to send Discord notification:', error);
  }
}
```

これを先ほどのジョブに組み込みます。

```typescript:pages/api/cron/safe-article-generation.ts (修正版)
import { NextApiRequest, NextApiResponse } from 'next';
import { acquireLock, releaseLock } from '../../lib/jobLock';
import { generateArticleContent } from '../../lib/articleGenerator';
import { sendDiscordNotification } from '../../lib/notification'; // 追加

const LOCK_KEY = 'article_generation_lock';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method Not Allowed' });
  }

  const authHeader = req.headers['authorization'];
  if (!authHeader || authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return res.status(401).json({ message: 'Unauthorized' });
  }

  let lockAcquired = false;
  try {
    lockAcquired = await acquireLock(LOCK_KEY, 120);

    if (!lockAcquired) {
      const message = '⚠️ Article generation skipped: Another instance is already running.';
      console.log(message);
      await sendDiscordNotification(message); // 通知
      return res.status(200).json({ message: 'Another instance is running, skipped.' });
    }

    console.log('Lock acquired. Starting article generation...');
    const theme = 'AIとWeb開発の最新トレンド';
    const articleContent = await generateArticleContent(theme);
    // ... 記事の保存やデプロイ処理 ...
    console.log('Article generation completed.');
    await sendDiscordNotification('✅ Article generation completed successfully for theme: ' + theme); // 成功時も通知

    return res.status(200).json({ message: 'Article generation process completed successfully.' });

  } catch (error: any) { // エラーの型をanyとして扱うか、より具体的な型アサーションを行う
    console.error('🚨 Error in safe article generation:', error);
    const errorMessage = `🚨 Article generation failed! Error: ${error.message || 'Unknown error'}`;
    await sendDiscordNotification(errorMessage); // エラー通知
    return res.status(500).json({ message: 'Failed to complete article generation.' });
  } finally {
    if (lockAcquired) {
      await releaseLock(LOCK_KEY);
      console.log('Lock released.');
    }
  }
}
```

これにより、ジョブの成功・失敗をリアルタイムで把握し、問題発生時には即座に対応できる体制を整えています。特にエラーログは、どのようなエラーが何時発生したのか、原因究明に役立つ情報をしっかり含めるようにしています。

### 課題4: Vercel Cronの認証とセキュリティ

Vercel CronのAPI Routeは、パブリックなエンドポイントとしてデプロイされます。もし認証なしで誰でもアクセスできる状態だと、悪意のある攻撃や意図しないトリガーにつながる可能性があります。

**解決策: CRON_SECRETによる認証**

Vercel Cronは、Cronジョブをトリガーする際に`Authorization`ヘッダーに`Bearer <CRON_SECRET>`という形式のシークレットを自動的に付与します。このシークレットはVercelプロジェクトの環境変数として設定し、API Route側で検証することで、Vercel Cronからの正規のリクエストであることを確認できます。

上記のコード例にも示していますが、再度強調します。

```typescript
  // Vercel Cronからの認証
  const authHeader = req.headers['authorization'];
  if (!authHeader || authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return res.status(401).json({ message: 'Unauthorized' });
  }
```

このチェックを全てのCronジョブAPI Routeの冒頭に入れることで、セキュリティを確保しています。`CRON_SECRET`は複雑な文字列とし、厳重に管理することが重要です。

## ベストプラクティスのまとめ

*   **処理を細かく分割し、タイムアウトを防ぐ:** 重い処理は複数のCronジョブに分けたり、キューシステムを導入して非同期に処理を進める。
*   **分散ロックで並列実行の競合を回避:** Redisなどの分散ロック機構を利用し、同時に実行される可能性のあるジョブのリソース競合を防ぐ。
*   **厳格なエラーハンドリングと通知:** 全てのジョブで`try-catch`を徹底し、エラー発生時には詳細なログとDiscordなどのチャットツールへの通知を行う。
*   **CRON_SECRETでアクセスを認証:** Vercel Cronからの正規のリクエストであることを`Authorization`ヘッダーのシークレットで検証し、セキュリティを確保する。
*   **データドリブンな改善サイクル:** Google Search Console APIやGA4 APIと連携し、Cronジョブの結果を定量的に評価し、継続的に改善を行う。

## まとめ

Vercel Cronは、Next.jsアプリケーションにおいて強力な自動化ツールです。しかし、本番環境での安定運用には、タイムアウト、並列処理、エラーハンドリング、セキュリティといった課題への対策が不可欠です。

Futuristic Imagination LLCでは、これらの課題に対して具体的な解決策を導入し、AIオウンドメディア11サイトを自動運用できるまでに進化させました。私自身が「補給不要の自販機型SaaS」の実現を目標に掲げているように、徹底した自動化と効率化は、事業成長の鍵を握ると確信しています。

今回紹介したベストプラクティスが、皆さんのVercel Cronを活用した開発・運用の一助となれば幸いです。

今回紹介したような、WebサービスとAIの融合による事業拡張、そしてNext.jsとGenerative AI、Vercel Cronを活用した「一人で大きな価値を生み出す完全自動化」システムの構築代行も承っています。ご興味があれば、ぜひ一度お問い合わせください。

→ https://www.futuristicimagination.co.jp/service/

## 参考

*   [Vercel Cron Jobs Documentation](https://vercel.com/docs/concepts/solutions/cron-jobs)
*   [Upstash Redis](https://upstash.com/)
*   [Discord Webhooks](https://discord.com/developers/docs/resources/webhook)