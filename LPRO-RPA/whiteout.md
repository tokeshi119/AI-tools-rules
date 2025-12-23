# inviteProviderJp における画面表示・ログイン関連の問題

[LINESPE-633](https://svsprj.backlog.jp/view/LINESPE-633) ホワイトアウトの件について


**調査日:** 2024年12月14日（初版）、2024年12月15日（追記・解決）  

**ステータス:** ✅ 解決済み（個人で進めただけです。まだ、PRだして修正はしてにない。あくまでfeature/searchで修正しただけ）

  

---

  

## 📋 概要

  ビズアカ作成時、プロバイダ招待時に発生する模様。
  前提：
  ＶＰＮなし
  手動だと問題ない
  LINE連携時は今まで１度も発生な
  し[https://account.line.biz/profile/account/verification](https://account.line.biz/profile/account/verification)⇒[https://developers.line.biz/console/](https://developers.line.biz/console/)の遷移時に確認できた  
　体感２～３回に１度程発生　他の部分でもあるかも

という報告

---

ここから自分での調査

`inviteProviderJp` RPA実行時に発生する2つの関連問題についての調査と対策。

  

| 問題 | 原因 | 発生条件 |

|------|------|----------|

| 問題1: 画面が真っ白のまま進まない | ページロード未完了 | ネットワーク遅延・マシン負荷時 |

| 問題2: ログインページ待ちがタイムアウト | ログイン済みセッション | 既存セッションがある時 |

  

---

  

## 🔴 問題1: 画面が真っ白のまま進まない（NEW）

  

### 発生状況

  

Chrome起動後、`DEV_HOME_URL` にアクセスするが、リダイレクトが発生せず画面が真っ白のまま次の処理に進まない。

  

### 原因分析

  

**ページロード完了を待つ処理が存在しない。**

  

```

inviteProviderJp (102-110行目)

    │

    ├─① launchChrome で DEV_HOME_URL を開く

    │    → chrome-launcher がChromeを起動するだけ

    │    → ページロード完了を待たない ❌

    │

    ├─② fetchBrowserInstance() で Playwright 接続

    │

    ├─③ getFirstTab() でタブ取得

    │    → この時点でページがまだロード中の可能性

    │

    └─④ loginUseCaseForDeveloper() を呼び出し

         │

         └─❌ ページが真っ白 or リダイレクト未完了でエラー

```

  

### 該当コード

  

**inviteProviderJp.ts (102-110行目)**

```typescript

const chromeResult: LaunchedChrome = await launchChrome({

    startingUrl: SITE.DEV_HOME_URL as string,

});

if (typeof chromeResult["pid"] === "undefined") {

    throw new RenderMessageError(

        "chromeが既に存在しています。chromeを閉じてください。",

    );

}

const tab = getFirstTab(await fetchBrowserInstance());

// ← ここにページロード待ちがない

await loginUseCaseForDeveloper(tab, ...);

```

  

### 影響範囲調査

  

プロジェクト全体で `waitForLoadState` / `waitForNavigation` を検索した結果：**使用箇所なし**

  

以下のRPAファイルすべてで同様のパターン（ページロード待ちなし）が使用されている：

  

- `inviteProviderJp.ts`

- `createLineOfficialAccountJp.ts`

- `createLineOfficialAccountKr.ts`

- `createLineOfficialAccountTh.ts`

- `createLineOfficialAccountEn.ts`

- `createLineOfficialAccountKrComu.ts`

- `createLineOfficialAccountThComu.ts`

- `createNewBusinessAccount.ts`

- `verifyTelephoneAuthCode.ts`

- `deleteTelephoneAuthCodeJp.ts`

- `deleteLineAccount.ts`

- `deleteBusinessAccount.ts`

- `editLineAccount.ts`

- `addGroup.ts`

  

**現時点で他のRPAでは問題は発生していないが、潜在的なリスクあり。**

  

---

  

## 🔴 問題2: セッション起因でログインページ待ちがタイムアウト

  

### 発生したエラー

  

```

page.waitForURL: Timeout 10000ms exceeded.

=========================== logs ===========================

waiting for navigation until "load"

============================================================

    at LoginPage.waitForOpenPage

    at async loginUseCaseForDeveloper

    at async inviteProviderJp

```

  

### 原因分析

  

```

inviteProviderJp (102-118行目)

    │

    ├─① launchChrome で DEV_HOME_URL を開く

    │    URL: https://developers.line.biz/console/

    │

    └─② loginUseCaseForDeveloper を呼び出し

         │

         └─③ LoginPage.waitForOpenPage を実行

              │

              └─❌ ログインページを10秒待つがタイムアウト

```

  

### 問題の本質

  

| 項目 | 想定（未ログイン時） | 実際（ログイン済み時） |

|------|----------------------|------------------------|

| 開始URL | `https://developers.line.biz/console/` | 同左 |

| リダイレクト先 | `https://account.line.biz/login?...` | リダイレクトなし |

| 結果 | ログインページで正常待機 | コンソールのまま → **タイムアウト** |

  

### 該当コード

  

**LoginPage.ts (18-24行目)**

```typescript

static async waitForOpenPage(page: Page): Promise<void> {

    console.log("LoginPage.waitForOpenPage current URL:", await page.url());

    await page.waitForURL(

        /https:\/\/account\.line\.biz\/login\?redirectUri=.*/,

        { timeout: WAIT_CONFIG.ELEMENT_TIMEOUT },  // 10秒

    );

}

```

  

### 切り分けログ

  

`waitForOpenPage` 冒頭の `console.log` で確認：

  

```

LoginPage.waitForOpenPage current URL: https://developers.line.biz/console/

```

  

→ ログインページにリダイレクトされておらず、コンソールに直接入っていることを確認済み。

  

---

  

## 🔧 現在の実装状況

  

| 項目 | 状態 |

|------|------|

| `launchChrome` | `startingUrl`/`port` のみ指定。`--user-data-dir` 等プロファイル指定なし |

| ページロード待ち | **なし**（`waitForLoadState` 未使用） |

| セッション管理 | `cookie`/`session` を操作するコードは見当たらない |

| ログイン状態チェック | **なし**（常にログインが必要な前提） |

  

※ `userDataDir` を指定するクリーンプロファイル起動（Phase 1.5）は、現行の `inviteProviderJp` では一旦オフ。白画面が再発したら再度有効化する。

  

---

  

## 💡 段階的対策プラン

  

### Phase 1: 最小限の対応（inviteProviderJp のみ）

  

**目的:** 影響範囲を限定して検証

  

**対象ファイル:** `electron/library/rpa/jp/inviteProviderJp.ts`

  

**変更内容:**

```typescript

// Before (110行目付近)

const tab = getFirstTab(await fetchBrowserInstance());

await loginUseCaseForDeveloper(tab, ...);

  

// After

const tab = getFirstTab(await fetchBrowserInstance());

await tab.waitForLoadState('load');  // ← 追加

await loginUseCaseForDeveloper(tab, ...);

```

  

**同様に148-158行目付近のループ内でも追加が必要**

  

| メリット | デメリット |

|----------|------------|

| 影響範囲が最小 | 他のRPAには適用されない |

| 検証が容易 | 同様の問題が他で発生する可能性は残る |

  

**検証項目:**

- [ ] Chrome起動後、画面が白いまま止まらないこと

- [ ] ログインページへの正常遷移

- [ ] 招待処理の正常完了

  

---

  

### Phase 2: 推奨対応（loginUseCase への適用）

  

**目的:** ログイン処理を使う全RPAで対応

  

**対象ファイル:** `electron/library/use_case/loginUseCase.ts`

  

**変更内容:**

```typescript

export const loginUseCaseForDeveloper = async (

    page: Page,

    businessAccount: BusinessAccount,

    imageVerification: ImageVerification,

): Promise<void> => {

    // ページロード完了を待つ

    await page.waitForLoadState('load');

    await LoginPage.waitForOpenPage(page);

    // ... 以降のログイン処理

};

  

// loginUseCase にも同様に追加

export const loginUseCase = async (

    page: Page,

    businessAccount: BusinessAccount,

    imageVerification: ImageVerification,

): Promise<void> => {

    // ページロード完了を待つ

    await page.waitForLoadState('load');

    await LoginPage.waitForOpenPage(page);

    // ... 以降のログイン処理

};

```

  

| メリット | デメリット |

|----------|------------|

| 全ログイン処理で一貫して対応 | 影響範囲が広い（全RPA） |

| 将来のリスク軽減 | テスト範囲が広がる |

  

**Phase 1 で問題が解決したら、Phase 2 に移行する。**

  

---

  

### Phase 3: 理想的な対応（将来検討）

  

**ユーティリティ関数として共通化**

  

```typescript

// rpa_utility.ts に追加

export const fetchBrowserInstanceWithWait = async (): Promise<Browser> => {

    const browser = await fetchBrowserInstance();

    const page = getFirstTab(browser);

    await page.waitForLoadState('load');

    return browser;

};

```

  

---

  

## ✅ 実施状況

  

| Phase | ステータス | 実施日 | 備考 |

|-------|------------|--------|------|

| Phase 1 | ✅ 解決済み | 2024-12-15 | inviteProviderJp のみ |

| Phase 1.5 | ✅ 解決済み | 2024-12-15 | userDataDir + sleep 追加 |

| Phase 1.6 | ✅ 解決済み | 2024-12-15 | retryConditionパターン導入 |

| Phase 2 | 🔲 未実施 | - | 必要に応じて実施 |

| Phase 3 | 🔲 検討中 | - | 将来的に検討 |

  

### Phase 1 変更内容（ページロード待ち）

  

**ファイル:** `electron/library/rpa/jp/inviteProviderJp.ts`

  

**変更箇所1:** 122-124行目（最初のChrome起動後）

```typescript

const tab = getFirstTab(await fetchBrowserInstance());

// ページロード完了を待つ（Phase 1: 画面が真っ白になる問題の対策）

await tab.waitForLoadState("load");

```

  

**変更箇所2:** 67-69行目（executeInviteRPA内）

```typescript

const tabOne: Page = getFirstTab(browser);

// ページロード完了を待つ（Phase 1: 画面が真っ白になる問題の対策）

await tabOne.waitForLoadState("load");

```

  

### Phase 1.5 変更内容（クリーンプロファイルで起動）

  

**問題:** デフォルトChromeプロファイルを使用するため、手動確認等でログインした状態が残っているとログインページにリダイレクトされない

  

**対策:** `userDataDir` を一時ディレクトリに指定してクリーンなプロファイルで起動

  

**現状:** `inviteProviderJp` では一旦 `userDataDir` 指定を外してデフォルトプロファイルで起動中。白画面が再発した場合は以下を再適用する。

  

**ファイル1:** `electron/library/rpa/launchChrome.ts`

```typescript

// userDataDir オプションを追加

export const launchChrome = async (

    options: { startingUrl?: string; port?: number; userDataDir?: string } = {},

): Promise<LaunchedChrome> => {

    const { startingUrl = "https://google.com", port = 9222, userDataDir } = options;

    return await launch({ startingUrl, port, userDataDir });

};

```

  

**ファイル2:** `electron/library/rpa/jp/inviteProviderJp.ts`

```typescript

// 一時ディレクトリ生成関数を追加

const createTempUserDataDir = (): string =>

    path.join(os.tmpdir(), `chrome-rpa-${Date.now()}`);

  

// launchChrome 呼び出し時に userDataDir を指定（2箇所）

const chromeResult = await launchChrome({

    startingUrl: SITE.DEV_HOME_URL as string,

    userDataDir: createTempUserDataDir(),

});

  

// 現行挙動（userDataDir を外してデフォルトプロファイルで起動する場合）

const chromeResult = await launchChrome({

    startingUrl: SITE.DEV_HOME_URL as string,

});

```

  

### Phase 1.6 変更内容（retryConditionパターン導入）

  

**問題:** LINE側のセッション管理が不安定で、クリーンプロファイルで起動してもランダムに「画面が真っ白」になることがある

  

**原因分析:**

- URLは`developers.line.biz/console`だが、ページが正常にロードされていない（サイドバーが表示されない）

- LINE側がIPベースやブラウザフィンガープリントでセッションを管理している可能性

- 数回リトライすると正常に表示されるパターンが観測された

  

**対策:** `retryCondition`パターンを導入し、ページが正常に表示されるまでリロードしてリトライ

  

**処理フロー:**

```

┌─────────────────────────────────────────────────────────────────┐

│  retryCondition（最大3回リトライ）                               │

├─────────────────────────────────────────────────────────────────┤

│  1. waitForLoadState("networkidle") でリダイレクトを待つ        │

│  2. URLチェック                                                  │

│     ├─ ログインページ → true（成功）                            │

│     └─ コンソールページ                                         │

│          ├─ サイドバー表示あり → true（成功、ログイン済み）     │

│          └─ サイドバー表示なし → reload → false（リトライ）    │

└─────────────────────────────────────────────────────────────────┘

```

  

**実装コード:**

```typescript

// ページが正常に表示されるまでリトライ（LINE側のセッション不安定対策）

await retryCondition(async (): Promise<boolean> => {

    await tab.waitForLoadState("networkidle");

    const currentUrl = tab.url();

  

    if (currentUrl.includes("account.line.biz/login")) {

        return true;

    }

  

    if (currentUrl.includes("developers.line.biz/console")) {

        try {

            await tab.locator("#sidebar").waitFor({ state: "visible", timeout: 4000 });

            return true;

        } catch {

            await tab.reload();

            return false;

        }

    }

  

    await tab.reload();

    return false;

});

  

// URLをチェックしてログイン処理を分岐

const currentUrl = tab.url();

if (currentUrl.includes("account.line.biz/login")) {

    await loginUseCaseForDeveloper(...);

}

```

  

**参考にしたパターン:** `AccountPageProfilePage.waitForOpenPageWithRetry`

  

**テスト結果:**

- 10回以上の連続実行で安定動作を確認

- リトライが発動したケースでも正常に回復することを確認

  

---

  

## 🔬 調査・解決プロセスの記録

  

この問題の解決に至るまでのプロセスを記録する。今後同様の問題が発生した場合の参考として。

  

### 調査の流れ

  

```

問題発生

    ↓

Step 1: 現象の確認

    - 「画面が真っ白のまま進まない」という報告

    - エラーログの収集

    ↓

Step 2: コードベースの調査

    - 関連ファイルの特定（inviteProviderJp.ts, loginUseCase.ts, launchChrome.ts）

    - 処理フローの把握

    - 類似パターンの検索（grep で waitForLoadState, launchChrome 等を検索）

    ↓

Step 3: 原因の仮説立て

    - 仮説1: ページロード完了を待っていない

    - 仮説2: ログイン済みセッションが残っている

    ↓

Step 4: 段階的な修正と検証

    - Phase 1: waitForLoadState 追加

    - Phase 1.5: userDataDir でクリーンプロファイル起動

    - 問題発生 → デバッグログ追加 → 原因特定 → 修正

    ↓

解決

```

  

### デバッグ手法

  

#### 1. 処理フローの可視化

  

問題箇所を特定するために、処理の要所に `console.log` を追加：

  

```typescript

// 例: inviteProviderJp.ts 内

console.log("[DEBUG] inviteMail 開始");

await ConsoleProviderRolePage.inviteMail(tab, invitedAccounts.length);

console.log("[DEBUG] inviteMail 完了");

chromeResult.kill();

console.log("[DEBUG] Chrome終了");

```

  

#### 2. ログの解釈

  

| ログパターン | 意味 |

|-------------|------|

| `[DEBUG] A` のみ表示 | A の後の処理で問題発生 |

| `[DEBUG] A` → `[DEBUG] B` | A〜B間は成功、B以降で問題 |

| エラー前の最後のログ | 問題箇所の直前 |

  

#### 3. 段階的なログ追加

  

最初は大きな単位でログを追加し、問題箇所が絞れたら詳細なログを追加：

  

```

Phase 1: 大きな処理単位

├─ [DEBUG] inviteMail 開始/完了

└─ [DEBUG] Chrome終了

  

Phase 2: 詳細な処理単位（問題箇所が絞れた後）

├─ [DEBUG] executeInviteRPA 開始

├─ [DEBUG] ページロード完了

├─ [DEBUG] ログイン完了

├─ [DEBUG] メール取得完了

├─ [DEBUG] 招待URL: {URL}

├─ [DEBUG] 招待URLに遷移完了

└─ [DEBUG] 招待承諾完了

```

  

### 発見された追加問題と対策

  

調査中に当初の問題以外にも問題が発見された：

  

| 問題 | 発見のきっかけ | 原因 | 対策 |

|------|--------------|------|------|

| 一時ディレクトリ未作成 | `ENOENT` エラー | `chrome-launcher` は `userDataDir` が存在する必要がある | `fs.mkdirSync` でディレクトリを事前作成 |

| Chrome終了後に次のChromeが起動できない | `chromeが既に存在しています` エラー | `kill()` 後にChromeプロセスが完全終了する前に次を起動 | `sleep(2000)` で待機時間追加 |

  

### 最終的な修正内容

  

#### 1. ページロード待ち（Phase 1）

  

```typescript

const tab = getFirstTab(await fetchBrowserInstance());

await tab.waitForLoadState("load");  // ← 追加

```

  

#### 2. クリーンプロファイル起動（Phase 1.5）

  

```typescript

// 一時ディレクトリ生成関数（タイムスタンプ + ランダム文字列で一意性を確保）

const createTempUserDataDir = (): string => {

    const uniqueId = `${Date.now()}-${Math.random().toString(36).substring(2, 8)}`;

    const tmpDir = path.join(os.tmpdir(), `chrome-rpa-${uniqueId}`);

    if (!fs.existsSync(tmpDir)) {

        fs.mkdirSync(tmpDir, { recursive: true });

    }

    return tmpDir;

};

  

// launchChrome 呼び出し時に指定

const chromeResult = await launchChrome({

    startingUrl: SITE.DEV_HOME_URL as string,

    userDataDir: createTempUserDataDir(),

});

```

  

#### 3. Chrome終了後の待機

  

```typescript

chromeResult.kill();

await sleep(2000);  // ← 追加（Chromeプロセス完全終了を待つ）

```

  

#### 4. retryConditionパターン（Phase 1.6）

  

```typescript

// ページが正常に表示されるまでリトライ（LINE側のセッション不安定対策）

await retryCondition(async (): Promise<boolean> => {

    await tab.waitForLoadState("networkidle");

    const currentUrl = tab.url();

  

    if (currentUrl.includes("account.line.biz/login")) {

        return true;  // ログインページに遷移した → 正常

    }

  

    if (currentUrl.includes("developers.line.biz/console")) {

        try {

            await tab.locator("#sidebar").waitFor({ state: "visible", timeout: 4000 });

            return true;  // サイドバー表示あり → ログイン済み、正常

        } catch {

            await tab.reload();  // サイドバーなし → 真っ白い画面、リロード

            return false;

        }

    }

  

    await tab.reload();

    return false;

});

```

  

### 教訓・ベストプラクティス

  

1. **段階的なアプローチ**

   - 最小限の修正から始め、検証しながら拡大する

   - 影響範囲を限定することでリスクを軽減

  

2. **デバッグログの活用**

   - 問題箇所の特定には `console.log` が効果的

   - 大きな単位から詳細へと段階的に追加

   - **解決後はデバッグログを削除してコードをクリーンに保つ**

  

3. **エラーメッセージの注意深い読み取り**

   - `ENOENT`: ファイル/ディレクトリが存在しない

   - `Target page, context or browser has been closed`: ブラウザが閉じられている

   - `Timeout exceeded`: 待機時間超過

  

4. **外部ライブラリの挙動理解**

   - `chrome-launcher`: `userDataDir` 未指定時はデフォルトプロファイルを使用

   - `kill()` は非同期的にプロセスを終了（即座には終了しない）

  

5. **ドキュメント化の重要性**

   - 調査プロセスを記録することで、同様の問題発生時の参考になる

   - 修正内容と理由を明記することで、将来のメンテナンスが容易に

  

6. **既存パターンの活用（retryCondition）**

   - プロジェクト内で既に使用されているパターンを参考にする

   - `AccountPageProfilePage.waitForOpenPageWithRetry` が参考になった

   - 外部サービス（LINE）の不安定さに対応するにはリトライ機構が有効

  

7. **コードの最適化**

   - 冗長なコードは保守性を下げる

   - 無駄な処理がないか確認し、シンプルに保つ

   - 例: `networkidle` の待機を `retryCondition` 内に集約

  

---

  

## 📁 関連ファイル

  

- `electron/library/rpa/jp/inviteProviderJp.ts`

- `electron/library/use_case/loginUseCase.ts`

- `electron/library/page_object/jp/manager/Login/LoginPage.ts`

- `electron/library/utility/rpa_utility.ts`

- `electron/library/rpa/launchChrome.ts`

- `electron/library/main_enum.ts`