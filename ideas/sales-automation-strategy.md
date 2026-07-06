---
created: "2026-03-20"
status: in-progress
tags: [自動化, 営業, 士業, 最重要]
---

# 営業メール自動化戦略（士業ターゲット）

## 概要
士業（税理士・社労士・行政書士・司法書士・弁護士など）を対象に、
① リストアップの自動化 → ② HPの問い合わせフォーム自動送信
の2段階で営業を自動化する。

---

## ⚠️ 最初に確認すべきこと（法的リスク）

### 特定電子メール法（迷惑メール防止法）
- **メールアドレスへの一斉送信**は、オプトインなしでは違法になりうる
- ただし「問い合わせフォーム」経由は現時点でグレーゾーン（メール送信ではないため）
- 問い合わせフォームの利用規約に「営業目的禁止」と書かれている場合は違反リスクあり

### 現実的な対応策
- ターゲットを絞って**半自動（リストは自動、送信は手動確認）**にする
- 1社1社の内容を少しカスタマイズして「明らかな一括送信」を避ける
- 送信量: 1日10〜30件程度に抑える（大量送信はブラックリスト入りリスク）

---

## STEP 1: 士業リストアップの自動化

### 取得すべき情報
- 事務所名
- 代表者名
- 所在地（都道府県・市区町村）
- HP URL
- 問い合わせフォームURL（または電話番号）
- 規模感（スタッフ数・設立年など）

### 方法A: Google Maps API（推奨・合法）

```python
# Google Places API を使ったリストアップ例
import requests

API_KEY = "your_google_api_key"
query = "税理士事務所 東京"

url = f"https://maps.googleapis.com/maps/api/place/textsearch/json"
params = {
    "query": query,
    "language": "ja",
    "key": API_KEY
}

response = requests.get(url, params=params)
places = response.json()["results"]

for place in places:
    print(place["name"], place.get("formatted_address"), place.get("website"))
```

**メリット**: 合法・安定・住所や評価も取得可能
**デメリット**: 有料（月$200の無料枠あり）、HP URLが取れないことがある

---

### 方法B: Playwright でのスクレイピング（準合法・グレーゾーン）

各士業の検索サイトからリストを取得する。

**主な取得元サイト:**
| 士業 | サイト |
|------|-------|
| 税理士 | 税理士ドットコム、Googleマップ |
| 社労士 | 社労士ドットコム、各都道府県会HP |
| 行政書士 | 行政書士ドットコム、Googleマップ |
| 司法書士 | 司法書士.com、Googleマップ |
| 弁護士 | 弁護士ドットコム |

```python
# Playwright を使った基本的なスクレイピング例
from playwright.async_api import async_playwright
import asyncio
import csv

async def scrape_offices(url):
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto(url)

        # 事務所名・URLを取得（サイトごとにセレクタ変更が必要）
        offices = await page.query_selector_all(".office-card")
        results = []
        for office in offices:
            name = await office.query_selector(".name")
            link = await office.query_selector("a")
            results.append({
                "name": await name.inner_text(),
                "url": await link.get_attribute("href")
            })

        await browser.close()
        return results
```

---

### 方法C: Google 検索の自動化（シンプル）

```python
# googlesearch-python ライブラリを使う例
from googlesearch import search
import time

queries = [
    "税理士事務所 東京 お問い合わせ",
    "社労士事務所 大阪 サイト:*.jp",
]

urls = []
for q in queries:
    for url in search(q, num_results=20, lang="ja"):
        urls.append(url)
    time.sleep(2)  # レートリミット対策
```

---

### リストの保存・管理

```python
# CSVで管理
import csv

fields = ["事務所名", "代表者名", "都道府県", "HP_URL", "フォームURL", "ステータス", "送信日"]

with open("sales_list.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=fields)
    writer.writeheader()
```

ステータス管理: `未送信` / `送信済み` / `返信あり` / `NG`

---

## STEP 2: 問い合わせフォーム自動入力

### 基本的な流れ

```
HPのURL → フォームを検出 → 各フィールドを入力 → 送信
```

### Playwright による自動フォーム入力

```python
from playwright.async_api import async_playwright
import asyncio

async def send_inquiry(url, company_name, message):
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)  # 確認用にheadless=False
        page = await browser.new_page()
        await page.goto(url)

        # 問い合わせページを探す
        contact_links = await page.query_selector_all("a:has-text('お問い合わせ'), a:has-text('contact')")
        if contact_links:
            await contact_links[0].click()
            await page.wait_for_load_state()

        # フォームに入力（フィールド名はサイトによって異なる）
        await page.fill("input[name='company'], input[name='会社名']", company_name)
        await page.fill("input[name='name'], input[name='お名前']", "あなたの名前")
        await page.fill("input[name='email'], input[name='メール']", "your@email.com")
        await page.fill("textarea[name='message'], textarea[name='お問い合わせ内容']", message)

        # ここで一時停止して人間が確認・送信（半自動）
        print(f"フォーム入力完了: {url}")
        input("Enterキーで送信...")

        # await page.click("button[type='submit']")  # 完全自動化する場合
        await browser.close()

# 実行
asyncio.run(send_inquiry("https://example-jimusho.com/contact", "株式会社〇〇", "営業メッセージ"))
```

---

## STEP 3: メッセージテンプレート

### 効果的な営業文の構成

```
件名: [事務所名]様のWEBからの新規顧客獲得についてご提案

[代表者名]先生

突然のご連絡失礼いたします。
[自分の会社名・名前]と申します。

士業事務所専門のWEBマーケティングを支援しております。

■ 現在のお悩みに心当たりはありませんか？
・ホームページからの問い合わせが少ない
・どんな情報を発信すれば良いかわからない
・既存客の紹介だけに頼っている

弊社では[実績・事例]を通じて、
月〇件の問い合わせ増加を実現してきました。

まずは30分の無料相談をご提供しております。
ご興味があればご返信ください。

[連絡先]
```

### バリエーション作成のポイント
- 冒頭の「[士業の種類]の先生」部分をリストから自動挿入
- 実績数字は定期的に更新
- 件名A/Bテスト: 開封率を比較する

---

## 推奨する実装ステップ

### フェーズ1（今週）: リストを手動で作る
- [ ] Googleマップで「税理士事務所 [地域]」を検索
- [ ] 50社のリストをCSVで作成（手動でOK）
- [ ] 各HPの問い合わせフォームURLを記録

### フェーズ2（来週）: 半自動化
- [ ] Playwright で問い合わせページを自動検出
- [ ] フォーム入力を自動化（送信だけ手動確認）
- [ ] 1日10〜20件を目安に送信・追跡

### フェーズ3（2週間後）: 完全自動化
- [ ] スクレイピングでリスト収集を自動化
- [ ] 返信率・成約率をCSVで集計
- [ ] 効果的なメッセージパターンに絞り込む

---

## 必要な技術スタック

| ツール | 用途 | 難易度 |
|-------|------|-------|
| Python | 全体の処理 | 低 |
| Playwright | ブラウザ自動化・スクレイピング | 低〜中 |
| pandas | CSVデータ管理 | 低 |
| Google Places API | 事務所リスト収集 | 低 |
| Google Sheets | 営業進捗管理（共有しやすい） | 低 |

```bash
pip install playwright pandas requests google-api-python-client
playwright install chromium
```

---

## KPI・管理指標

| 指標 | 目標 |
|------|------|
| 送信数/日 | 10〜30件 |
| 開封・返信率 | 3〜5% |
| 月間返信数 | 15〜45件 |
| 成約率（返信→契約） | 10〜20% |
| 月間新規成約 | 2〜5件 |

---

## ネクストステップ
- [ ] まず50社のリストをGoogleマップから手動で作る
- [ ] 営業文のA/Bパターンを2〜3種類作る
- [ ] Playwright のサンプルコードを実際のサイトで動作確認する
