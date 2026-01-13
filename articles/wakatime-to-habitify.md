---
title: "Wakatime × Habitifyでコーディング習慣を自動記録する仕組みを作った" # 記事のタイトル
emoji: "👩‍💻" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["typescript", "habitify", "githubactions", "wakatime"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
弊社では、マーダーミステリーをプレイできるアプリ「ウズ」と、マーダーミステリーを制作しウズ上で利用できるアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[死者の告解](https://mdms.jp/scenarios/10276)」です。

⚠️今回は業務と全く関係ない話です。

## コーディングを習慣化したい
新年を迎え、新たな目標設定とともに「新しい習慣を身につけたい」と意気込んでいる方も多いのではないでしょうか。

私自身も「新しいプログラミング言語を習得したい」「何かサービスを作って公開したい」という目標を掲げ、業務時間外でのコーディングを習慣化したいと考えていました。

しかし、新年に立てた目標や習慣は、往々にして三日坊主で終わってしまうのが現実です。私もかつてはそうでした。
今年こそは習慣化を達成したい。そう思い習慣化についての知識をいろいろ調べてみると、「ハビットトラッカー」といって習慣化したい行動を行った時に記録することで習慣の継続をサポートする効果があるということが分かりました。

そこで、コーディング習慣の記録そのものを自動化し、楽にハビットトラッカーが続けられるように、Wakatime APIとHabitify APIを活用し、WakatimeとHabitifyを連携させてGitHub Actionsで毎日自動で同期する仕組みを構築しました

## wakatimeとは
https://wakatime.com/

Wakatimeは、コーディング時間を自動で計測し、使用したプログラミング言語、エディタ、プロジェクトごとの作業時間などを詳細に可視化してくれるサービスです。日々の開発活動を客観的なデータとして把握できるため、自身の生産性向上や学習計画の策定に役立ちます。VSCodeなど主要なIDEやエディタにプラグインとして導入でき、特別な操作なしに利用開始できる点が魅力です。


## habitifyとは
https://habitify.me/ja

Habitifyは、習慣の形成と継続をサポートする多機能な習慣トラッキングアプリです。目標とする習慣を設定し、日々の達成状況を記録することで、視覚的に進捗を把握できます。リマインダー機能や統計データによる分析機能も充実しており、モチベーションの維持や習慣の定着を強力に後押しします。複数のデバイス間でデータを同期できるため、いつでもどこでも習慣を管理できます。

## 作成したもの
勉強用のリポジトリを作成し、そのリポジトリでどれくらい作業したのかをWakatimeで計測したものを一日の終わりに取得し、Habitifyに同期するbotを作成しました。

ちょっとわかりにくいですが、一日の終わりにこのように特定のリポジトリを触った時間が自動で追加されます。
![記録](</images/wakatime-to-habitify-1.png>)

このシステムを構築するためのステップは以下の通りです。

## wakatimeのプラグインをインストール　
まずはwakatimeのアカウント登録を行い、その後 https://wakatime.com/settings/account からAPIキーを発行します。
その後、VSCodeをお使いの方はExtensionからwakatimeと検索すればすぐ出てくるのでインストールし、先ほど発行したAPIキーを入力します。
![alt text](</images/wakatime-to-habitify-2.png>)

## habitifyで習慣を作成
続いてhabitifyをインストールします。PC版もありますが、なぜかAPIキーの発行がモバイル版しかできないので、モバイルアプリをインストールしましょう。　

インストール後は、対象となる習慣を作成しましょう。この時、「目標」の箇所を編集し、単位を「分」にしてください。これによりwakatime側でトラッキングした時間をそのまま流しこむことができます。

![alt text](</images/wakatime-to-habitify-3.png>)
![alt text](</images/wakatime-to-habitify-4.png>)


## スクリプトを実装
今回はtypescriptで実装しました。コード全文は [こちら。](https://github.com/piesuke/habitify)

抜粋して解説します。

### 1.wakatimeの実装
```typescript
// src/lib/wakatime.ts
import { WAKATIME_API_KEY } from "../constant";

type WakaSummariesResponse = {
	data?: Array<{
		projects?: Array<{ name: string; total_seconds: number }>;
		grand_total?: { total_seconds: number };
	}>;
};

/**
 * 当日の特定のプロジェクトの編集時間を取得する
 * 
**/
export const getTodayTimeForProject = async (
	project: string,
	date: string, // YYYY-MM-DD
) => {
	const url =
		`https://wakatime.com/api/v1/users/current/summaries` +
		`?start=${encodeURIComponent(date)}` +
		`&end=${encodeURIComponent(date)}` +
		`&project=${encodeURIComponent(project)}`;

	const res = await fetch(url, {
		headers: {
			Authorization: `Basic ${Buffer.from(`${WAKATIME_API_KEY}:`).toString("base64")}`,
		},
	});
	if (!res.ok) {
		throw new Error(
			`Failed to fetch Wakatime data: ${res.status} ${res.statusText}`,
		);
	}
	const data = (await res.json()) as WakaSummariesResponse;

	const totalSeconds = data.data?.[0]?.grand_total?.total_seconds || 0;
	return totalSeconds;
};
```

### 2.habitifyの実装

```typescript
// src/lib/habitify.ts

/**
 * 特定の習慣のidを引数にとり、その習慣に分数のログを追加する
 * 
**/
export const updateHabitifyMinLog = async (
	habitId: string,
	min: number,
	date: string, // YYYY-MM-DDTHH:MM:SS+timezone
) => {
	const url = `https://api.habitify.me/logs/${encodeURIComponent(habitId)}`;

	const res = await fetch(url, {
		method: "POST",
		headers: {
			"Content-Type": "application/json",
			Authorization: `${process.env.HABITIFY_API_KEY}`,
		},
		body: JSON.stringify({
			target_date: date,
			unit_type: "min",
			value: min,
		}),
	});
	if (!res.ok) {
		console.log(await res.text());
		throw new Error(
			`Failed to set Habitify status: ${res.status} ${res.statusText}`,
		);
	}
};

```



### 3.同期する実装
```typescript
// src/cmd/sync-wakatime-to-habitify.ts
import { HABITIFY_TARGET_HABIT_ID, WAKATIME_TARGET_PROJECT } from "../constant";
import { updateHabitifyMinLog } from "../lib/habitify";
import { getTodayTimeForProject } from "../lib/wakatime";

async function main() {
	console.log("Syncing Wakatime data to Habitify...");

	const today = new Date().toISOString().split("T")[0]; // YYYY-MM-DD\
	if (!today) {
		throw new Error("Failed to get today's date");
	}
	const totalSeconds = await getTodayTimeForProject(
		WAKATIME_TARGET_PROJECT,
		today,
	);
	const totalMinutes = Math.floor(totalSeconds / 60);

	const now = new Date();
	const jst = new Date(now.getTime() + 9 * 60 * 60 * 1000);
    // habitifyのAPIでは時間はYYYY-MM-DDTHH:MM:SS+timezone形式の必要がある。
	const timestamp = `${jst.toISOString().slice(0, 19)}+09:00`;

	await updateHabitifyMinLog(HABITIFY_TARGET_HABIT_ID, totalMinutes, timestamp);

	console.log("Sync completed.");
}

main().catch((err) => {
	console.error("Error during sync:", err);
	process.exit(1);
});

```

今回環境変数に保存した `HABITIFY_TARGET_HABIT_ID`, `WAKATIME_TARGET_PROJECT`はそれぞれ以下のような方法で取得します。

- HABITIFY_TARGET_HABIT_ID
  - https://docs.habitify.me/core-resources/habits#list-habits を参考に、https://api.habitify.me/habits を叩きそれぞれの習慣のIDを取得する。
- WAKATIME_TARGET_PROJECT
  - APIを叩くか、https://wakatime.com/projects でそれぞれのプロジェクト名を確認する。基本的にリポジトリ名と同じだが、たまに違うので注意。

### 4.github actionsでcron jobを回す実装
```yml
name: Sync Wakatime to Habitify

on:
  schedule:
    # 毎日23:55 JST (14:55 UTC)
    - cron: '55 14 * * *'
  workflow_dispatch: # 手動実行用

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install

      - name: Run sync
        run: pnpm run sync-wakatime-to-habitify
        # envのWAKATIME_TARGET_PROJECTとHABITIFY_TARGET_HABIT_IDを変更することで
        # 他のプロジェクトや習慣もトラッキング可能
        env:
          WAKATIME_API_KEY: ${{ secrets.WAKATIME_API_KEY }}
          HABITIFY_API_KEY: ${{ secrets.HABITIFY_API_KEY }}
          WAKATIME_TARGET_PROJECT: ${{ secrets.WAKATIME_TARGET_PROJECT }}
          HABITIFY_TARGET_HABIT_ID: ${{ secrets.HABITIFY_TARGET_HABIT_ID }}
```
先ほど紹介したwakatimeとhabitifyを同期するスクリプトをpacage.jsonに、

```json
"scripts": {
    "sync-wakatime-to-habitify": "tsx  src/cmd/sync-wakatime-to-habitify.ts"
  },
```
という形で登録し、それをgithub actionsで一日一回叩くようにしています。

## まとめ
自動トラッキングは出来ましたが、そもそもコーディング習慣を忘れるという事態も発生しそうなので、今後はHabitifyの記録をLINEやSlackなどに流すことによって、習慣化をより確実なものにしていきたいです。

また、今年はRustを習得したいので、このスクリプトをRustで書き直したいと思います。
