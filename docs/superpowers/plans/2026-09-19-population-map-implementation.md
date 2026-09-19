# 階層型世界人口地図 Implementation Plan

> **For agentic workers:** 実行時は superpowers:executing-plans を使用し、下記チェックボックスを順番に進める。ユーザーは設計判断の委任と事後報告を希望している。定型的な選択を再質問しない。本依頼の成果物は計画であり、この文書作成時点ではアプリ実装を開始しない。

**Goal:** iPadとNotionから、世界の地域別人口を国別、日本国内の市区町村別まで連続的に探索できる地図を実装する。

**Architecture:** MapLibreを地図操作、deck.glを比例円描画に使用する。階層制御と面積計算をライブラリから独立した純粋関数にし、公式データから生成した静的ファイルを親地域単位で読み込む。GitHub Pages上の既存教材と共存する独立ディレクトリへ公開する。

**Tech Stack:** TypeScript、Vite、MapLibre GL JS、deck.gl、Vitest、Playwright、Node.js、Python（統計・境界変換）、GitHub Pages。React、サーバー、ログインは導入しない。

**Spec:** [2026-09-13-population-map-design.md](../specs/2026-09-13-population-map-design.md)

**計画日:** 2026-09-19
**確認した仕様書blob SHA:** 95e870f5bc790e262916d1c3c0cdd2e5b1beb0c0
**確認したmainのtree SHA:** 566ab4aace84861697e7f20fbd844cd7a2b252b7
**対象:** fact0404/uk-sec-visual-learning-public
**状態:** Codexへの引き継ぎ用計画。以下のテストは実行予定であり、合格済みではない。

## Global Constraints

- 独立したWebアプリ
- 同じWebアプリのNotion埋め込み
- iPad横向き全画面を基準とするレスポンシブ表示
- 初期版の国内階層は日本だけを対象とする。
- 将来の優先拡張対象はOECD加盟国、国連安全保障理事会常任理事国などの主要国とし、各国の第一行政区画を追加する。
- 初期モード: 自由探索
- 選択表示: 地域名と人口だけ
- 初期指標: 人口のみ
- データ年: 世界・国は同一年、日本国内は最新の公式値
- 注釈: 通常画面では最小限
- 主要操作のタップ領域を44px以上にする
- 面積保存の判定は、2px未満のロケータードットへ置き換える前の比例半径に対して行う。
- 初期表示用のアプリコードと初期データ: gzip後1MB以内を目標
- 初期地図表示: 通常の学校・家庭回線で3秒以内を目標
- ズーム・分割中: 60fpsを目標、継続的に30fpsを下回らない

面積保存の適用範囲は、以下の「実装に必要な仕様補足」で明確化する。性能値は実測目標であり、未測定を合格扱いしない。

## Review Focus

1. 親人口と子合計が異なる場合、分割終了直前・直後に円が跳ねず、完了後は同一縮尺で実数値を比較できること。Task 4。
2. 日本から他地域へ移動した直後に古い通信応答が戻っても、古い地域を勝手に展開しないこと。Task 5・10。
3. 日付変更線をまたぐ地域、離島、極小国でも、円が地球を横切ったり詳細化不能になったりしないこと。Task 3・6・7。
4. 市総数と政令市の行政区、男女・年齢・国籍の内訳、国の地域別総数を同時加算しないこと。Task 2。
5. 小さい円の操作領域が重なっても選択が安定し、iframe内スクロールと地図操作が競合しないこと。Task 8・9。

---

## 0. 実行範囲とリポジトリの扱い

現行リポジトリは単体HTMLをルートへ並べた公開教材置き場。確認時点でpackage.json、AGENTS.md、.githubのワークフローはなく、人口地図は比較用HTML2枚と仕様書だけが存在する。実行開始時に再確認する。

実装元は `apps/population-map/`、ビルド出力の公開先は `population-map/` とする。仕様書内の `src/` と `public/` はアプリルートを基準に読み替える。ルートのindex.htmlを置換しない。

予定公開URL:
https://fact0404.github.io/uk-sec-visual-learning-public/population-map/

このURLは実装・配信確認前の予定であり、現時点で閲覧できるとは扱わない。

実装時の初動:

```bash
git status --short
git remote -v
git fetch origin
git ls-tree -r --name-only origin/main
git log -5 --oneline
```

AGENTS.mdがあれば親ディレクトリを含めて読む。既存チェックアウトに未コミット変更があれば、別worktreeへ切り出す。新規なら指定リポジトリをcloneし、`feat/population-map`ブランチを使用する。同名ブランチが既にあれば内容を調べて再開し、初期化し直さない。

各Taskを完成・検証して小さくコミットする。pushはセッションで許可済みだが、ブランチ保護や実行環境の承認を迂回しない。認証情報は文書・HTMLへ保存しない。

## 1. 実装に必要な仕様補足

ユーザーの「最善と思われる選択肢を選び、後で報告する」指示に基づく実装判断。原仕様の関連箇所と矛盾する場合は、この節を採用し、Task 1で原仕様にも反映する。

### 1.1 統計年差と面積保存

原仕様の式は親人口と子合計が一致する場合にだけ厳密な保存になる。描画補正を完了時に突然解除する実装は禁止する。

親の現在の描画人口をP、子の元人口合計をS、分割率をtとする。s(t)=3t²−2t³として、次を使用する。

```text
B(t) = (1 - s(t)) × P + s(t) × S
parentMass = (1 - t) × B(t)
childMass[i] = t × B(t) × rawChild[i] / S
radius[i] = commonScale × sqrt(mass[i])
```

- P=Sなら総面積を保存する。
- P≠Sなら総面積は差分を連続的に反映し、t=1で子の元人口へ一致する。
- S=0でP>0、欠損、不完全収録の場合は分割しない。
- P=S=0なら塗り円を描かず、地域選択で0人と表示できるようにする。
- 原仕様の受入4は「同一合計の分割で保存、異なる合計はB(t)に一致」へ修正する。
- 面積保存は同一フレームの共通縮尺kで面積をk²で割って検証する。ズーム補正や画面外クリップによる可視面積の変化を人口変化と混同しない。
- 子の実数値間の面積比保証は、分割が完了した円に適用する。分割中の親子円は遷移表現として扱う。
- 親が分割途中なら、その子は孫への分割を開始しない。同じ経路で二つの分割を同時進行させない。

### 1.2 六大州とM49

六大州はアジア、アフリカ、ヨーロッパ、北アメリカ、南アメリカ、オセアニア。M49の上位分類をそのまま六大州と同一視しない。

- 北アメリカはNorthern America、Central America、Caribbeanを含む。
- 南アメリカの次段が同じ範囲しか持たない場合は単一子を自動スキップし、国へ展開する。
- アフリカはM49の中間地域も使い、北・西・中部・東・南へ対応づける。
- 各国・地域はM49所属を基準に一つの親へ割り当てる。大陸横断国を形状で人口分割しない。
- 国際統計の国・地域単位を維持し、海外領土を含む総数と別建て地域の重複を除く。
- M49と人口統計の収録集合の差は手動対応表に根拠URL付きで記録する。独自IDが必要な場合は `stat-area:` 接頭辞を使い、ISOコードを捏造しない。
- 南極は背景だけ表示する。Web Mercatorでは極域が省略されるため、全大陸の形が完全に見えるとは約束しない。

### 1.3 日本の市区町村

都道府県の直下は市・町・村・東京都特別区。政令指定都市は市全体を1件とし、行政区は初期版の加算対象から除外する。郡や「市部」「町村部」などの集計行も除外する。境界が行政区別なら市へdissolveする。

当該基準日の自治体コード一覧を分母に収録件数を検証する。自治体総数を手書き定数にしない。合併・廃止でコードが変われば新エンティティとして履歴対応を残す。「固有IDは行政変更をまたいで不変」とはしない。

### 1.4 欠損

元仕様の「人口収録率98%」は欠損人口が不明だと評価できない。件数収録率を品質指標とし、公開初期版の国内詳細は期待自治体の100%を収録する。実行時の分割条件は、全子のID・有効値が揃うこと。欠損を0と置換せず、98%以上でも未収録子がある親は分割しない。

### 1.5 全画面と注釈

「全画面で開く」は同じ状態のトップレベルURLを新しいタブで開くリンクを基本とする。iframeのFullscreen API許可に依存しない。実際のNotionへの埋め込み操作は別途依頼がなければ行わない。

地図上の情報カードは地域名・人口のみ。出典・基準日・処理方法は小さな「データについて」から確認できる静的説明へ置く。データ利用条件で必要な帰属表示は省かない。自動統計更新や利用状況収集は追加しない。

## 2. ファイル構成と責務

以下はすべて新規予定。既存同名があれば比較して流用する。Pは文書内の略号 `apps/population-map` を表し、シェル変数ではない。

| パス（リポジトリ相対） | 責務 |
|---|---|
| apps/population-map/package.json、package-lock.json、tsconfig.json、vite.config.ts | 独立した開発・ビルド |
| apps/population-map/src/data/types.ts、schemas.ts、metric-registry.ts | 公開型、入力検証、指標定義 |
| apps/population-map/src/data/data-loader.ts、data-cache.ts | ファイル取得、再試行、版別キャッシュ |
| apps/population-map/data-input/source-lock.json、crosswalks.json | 実ファイルURL・SHA・列定義と地域対応 |
| apps/population-map/scripts/data/{acquire,normalize,boundaries,validate,build}.py | 原本取得、人口変換、境界変換、検証、書出し |
| apps/population-map/requirements.lock | データ生成用Python依存を固定 |
| apps/population-map/src/hierarchy/{detail-score,transition-state,hierarchy-controller}.ts | 詳細度、遷移状態、重複のない描画集合 |
| apps/population-map/src/symbols/{radius-scale,split-mass,collision-layout,hit-test}.ts | 数量と面積、衝突、選択の純粋計算 |
| apps/population-map/src/symbols/{proportional-circle-layer,leader-lines}.ts | deck.glとの接続 |
| apps/population-map/src/map/{map-controller,boundary-layer,viewport}.ts | MapLibre、境界、座標変換 |
| apps/population-map/src/app/{bootstrap,url-state,embed-mode}.ts | 初期化、URL、埋め込み |
| apps/population-map/src/ui/{controls,selection-card,labels,legend,format-population}.ts | UIと表示 |
| apps/population-map/src/main.ts、style.css、index.html | エントリー |
| apps/population-map/public/data/ | 生成データのみ |
| apps/population-map/public/data-about.html | 出典・日時・変換方針 |
| apps/population-map/tests/unit/、tests/data/、tests/e2e/ | 各Taskの意味のある検証 |
| apps/population-map/vitest.config.ts、playwright.config.ts | 検証環境 |
| apps/population-map/scripts/{check-size,stage-pages}.mjs | 転送量集計と公開先限定コピー |
| apps/population-map/README.md | 開発・更新・再開手順 |
| docs/superpowers/reports/population-map-acceptance.md | 実機・自動検証と残存制約 |
| .github/workflows/population-map-check.yml | このアプリだけの検証 |
| population-map/ | ビルド成果物の静的公開先 |

## 3. 共有インターフェース

Task 1で `P/src/data/types.ts` に次の型を定義する。後続Taskはこの名前を使い、ライブラリ型を混入させない。

```ts
export type GeoId = string;
export type XY = readonly [number, number];
export type Level = 'world' | 'continent' | 'subregion' | 'country' | 'admin1' | 'admin2';
export interface GeoNode {
  id: GeoId;
  parentId: GeoId | null;
  level: Level;
  name: {ja: string; en: string};
  anchor: XY;
  bounds: readonly [number, number, number, number]; // antimeridian: east may exceed 180
  childIds: GeoId[];
  childrenData: string | null;
  boundaryRef: string | null;
}
export interface Observation {
  geoId: GeoId;
  metricId: string;
  value: number | null;
  referenceDate: string;
  sourceId: string;
  valueStatus: 'estimate' | 'census' | 'register' | 'projection' | 'missing';
}
export interface ChildBundle {parentId: GeoId; nodes: GeoNode[]}
export interface Manifest {
  schemaVersion: 1;
  dataVersion: string;
  files: Record<string, {path: string; sha256: string; bytes: number}>;
  rootFile: string;
  observationsFile: string;
  countryIndexFile: string;
}
export interface PointSymbol {
  id: GeoId;
  parentId: GeoId | null;
  anchorPx: XY;
  centerPx: XY;
  rawValue: number;
  mass: number;
  radiusPx: number;       // proportional radius, before locator replacement
  locator: boolean;
  phase: 'stable' | 'parent' | 'child';
  t: number;
}
export interface MapState {
  lng: number; lat: number; zoom: number;
  selectedId: GeoId | null; metricId: 'population';
}
export interface Frame {
  symbols: PointSymbol[];
  requestedParents: GeoId[];
  scale: number;
}
```

純粋関数は単位テスト、通信はfetch注入、地図操作はブラウザ操作テストで検証する。以下のスニペットの人口は合成人口で、教材の実統計ではない。

## 4. 実行順序

Task 1→2→3→4→5→6→7→8→9→10→11→12で進む。Task 7終了で実地図と円を初めて接続する。Task 10で日本全国を含む初期版を通して動かす。大阪府だけの動作確認を全国完成と扱わない。

各Task内の一つのチェック項目を短い作業単位とする。大きい項目は列挙したファイルごとに実装・検証し、全項目が終わるまでTaskを完了にしない。

### Task 1: 開発基盤と実装仕様の固定

**Files:** P/package.json、P/package-lock.json、P/tsconfig.json、P/vite.config.ts、P/vitest.config.ts、P/src/data/types.ts、P/src/data/schemas.ts、P/src/data/metric-registry.ts、P/tests/unit/schemas.test.ts、P/README.md、P/.gitignore。Modify: 原仕様書。

**Interfaces:** `parseObservation(input: unknown): Observation`、`assertGeoTree(nodes: GeoNode[]): void`。不正入力はError。指標定義は `populationMetric` のみ実体化。

- [ ] 現行main、AGENTS.md、公開方法を確認し、原仕様に本計画1節を反映する。既存教材ファイルのSHA一覧を保存し、Task 12の差分確認に使う。
- [ ] P内にpackage.jsonを作る。Node 22.12以上の対応LTSを選び、実使用バージョンをREADMEと.nvmrcへ固定する。依存は対応する安定版を取得後にexact＋lockfileで固定する。deck.glの各packageは同一バージョンにする。試験用CDNやlatest URLを本体HTMLに残さない。
- [ ] 以下のscriptsを登録する。PythonはP内の.venvを作り、以後のPythonコマンドをその環境で実行する。

```json
{
  "scripts": {
    "dev": "vite --host 0.0.0.0",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "data:acquire": "python scripts/data/acquire.py",
    "data:build": "python scripts/data/build.py",
    "data:validate": "python scripts/data/validate.py",
    "build": "npm run data:validate && npm run typecheck && vite build",
    "preview": "vite preview --host 0.0.0.0 --port 4173",
    "test:e2e": "playwright test",
    "check:size": "node scripts/check-size.mjs",
    "pages:stage": "node scripts/stage-pages.mjs"
  }
}
```

- [ ] `schemas.test.ts` に次を追加し、`npm test -- tests/unit/schemas.test.ts`で未実装による失敗を確認する。

```ts
import {expect, test} from 'vitest';
import {parseObservation} from '../../src/data/schemas';
test('unknown and zero are distinct', () => {
  const base = {geoId:'test:a', metricId:'population',
    referenceDate:'2020-01-01', sourceId:'synthetic', valueStatus:'estimate'};
  expect(parseObservation({...base, value:0}).value).toBe(0);
  expect(parseObservation({...base, value:null, valueStatus:'missing'}).value).toBeNull();
  expect(() => parseObservation({...base, value:-1})).toThrow();
  expect(() => parseObservation({...base, value:NaN})).toThrow();
});
```

- [ ] 共有型・実行時検証を実装する。循環、欠けた親、world以外のnull親、ID重複を拒否する。worldは唯一のnull親として許可する。rateは将来の分子・分母定義を格納できる型だけ残し、単純平均を既定にしない。
- [ ] typecheckとschemas.testを通す。この時点では未作成のデータ生成・buildを成功扱いしない。`chore: scaffold isolated population map contracts`で対象ファイルだけコミットする。

### Task 2: 実統計の取得と重複のない人口集合

**Files:** P/scripts/data/acquire.py、normalize.py、validate.py、P/data-input/source-lock.json、crosswalks.json、P/requirements.lock、P/tests/data/test_population.py。

**Interfaces:** Python `normalize_world(rows, source) -> list[dict]`、`normalize_japan(rows, source) -> list[dict]`、`validate_population(nodes, observations) -> list[str]`。列定義はsource-lockに記録する。

- [ ] 公式配布ページから実ファイルを取得する。source-lockへ配布ページ、解決後URL、版、基準日、取得日、SHA256、文字コード、シート/列、単位、対象範囲、利用条件URLを記録する。原本は.venvと同様にGit対象から除外し、再取得可能なURLと小さい検証サンプルを残す。再配布が許可されサイズが適切なら原本を別の永続保管先に保存して参照を記録する。
- [ ] 最新年は「最も未来の列」を取らない。最新公開版で、取得日以前の基準日を持つ最新の共通年を使用する。推計/予測の区別は公式メタデータを保持する。未来予測しか得られない年を勝手に現在人口と呼ばない。
- [ ] 世界の国・地域はUN WPP、地域区分はM49。日本の都道府県は人口推計、市区町村は住民基本台帳の総人口（日本人＋外国人、男女計）を採用する。原本の千人単位は人数へ変換する。
- [ ] 次のテストを先に追加し、`python -m pytest tests/data/test_population.py -q`で失敗を確認する。ここに使うsourceの列名定義は合成入力用に固定する。

```python
from scripts.data.normalize import normalize_japan
def test_city_total_does_not_include_wards_twice():
    source = {"id":"synthetic", "date":"2020-01-01",
              "columns":{"code":"code","kind":"kind","population":"value"},
              "unit_multiplier":1}
    rows = [
        {"code":"27100","kind":"city","value":100},
        {"code":"27102","kind":"designated_city_ward","value":40},
        {"code":"27103","kind":"designated_city_ward","value":60},
        {"code":"13101","kind":"special_ward","value":25},
    ]
    out = normalize_japan(rows, source)
    assert {x["geoId"] for x in out} == {"jp-muni:27100","jp-muni:13101"}
    assert sum(x["value"] for x in out) == 125
```

- [ ] 国際集計行、男女別、年齢別、国籍別、市区町村小計を除外する。国・地域の包含関係はsource-lockの対象範囲と対応表で照合する。欠損記号を0にしない。
- [ ] 同じ基準日の全国自治体一覧とのID突合、47都道府県、各観測値の出典存在を検証する。欠損・重複・未知コードを一覧にし、解決するまで正式データ生成を失敗終了させる。
- [ ] テスト合格と実データ検証結果をコミットする。正式データが取得できなければ取得エラーを残し、合成fixtureで後続の計算・UI作業を進められるが、公開初期版の完了条件は満たさない。

### Task 3: 境界と階層データの生成

**Files:** P/scripts/data/boundaries.py、build.py、P/tests/data/test_boundaries.py、P/public/data/、P/public/data-about.html。Modify: crosswalks.json、source-lock.json。

**Interfaces:** `unwrap_bounds(longitudes: list[float]) -> tuple[float,float]`、`build_dataset(output_dir) -> dict`。出力は共有型に適合したManifest、ChildBundle、Observation[]。

- [ ] Natural Earthとe-Stat統計GISの境界を取得する。使用版、座標参照系、利用条件をsource-lockへ追記する。地理名照合で推測結合せず、公式コード対応を作る。
- [ ] 以下を `tests/data/test_boundaries.py` に追加して失敗確認後、日付変更線をまたぐ最短範囲を実装する。

```python
from scripts.data.boundaries import unwrap_bounds
def test_dateline_bounds_use_short_arc():
    west, east = unwrap_bounds([179.0, -179.0])
    assert abs((east - west) - 2.0) < 1e-8
```

- [ ] EPSG:4326へ変換する。町丁目境界を自治体コードで統合し、政令市は市単位に統合する。島・飛地を削除しない。共有境界の簡略化は同一レイヤーでまとめて行い、穴や隙間を作らない。
- [ ] 各ノードの表示用boundsとアンカーを事前計算する。境界なしでも詳細度判定が可能になる。アンカーは主要陸域の内部点を既定とし、島国・遠方領土は手動対応を許可する。六大州のアンカーは重なりにくい位置へ固定し、理由を対応表へ記す。
- [ ] 六大州→小地域→国の親子を作り、集計値を重複のない国・地域観測値から計算する。WPPの地域総数を別途加算しない。単一子を表示でスキップできる構造を保持する。
- [ ] 世界、国一覧の軽量index、日本の都道府県、都道府県別子bundleと境界を出力する。country-indexには経路解決に必要なID・親だけを含める。観測値は初期版で一括、地理・境界は段階読込。
- [ ] manifestには全ファイルの相対パス・ハッシュ・サイズ・データ版を記録する。同じ入力と同じ取得情報から同じ出力を生成する。ビルドの現在時刻だけで全ハッシュが変わらないよう、取得snapshot日時を使う。
- [ ] `python -m pytest tests/data -q`、`npm run data:build`、`npm run data:validate`を通す。data-aboutに統計年差と面積補正、2px点の意味、出典を短く記載する。
- [ ] 生成物・変換スクリプト・ロックをコミットする。以後通常buildは保存済み生成物を検証し、外部統計へ接続しない。

### Task 4: 面積計算と連続分割の純粋関数

**Files:** P/src/symbols/split-mass.ts、radius-scale.ts、P/tests/unit/split-mass.test.ts、radius-scale.test.ts。

**Interfaces:** `splitMass(parent: number, children: number[], t: number): {parent:number; children:number[]}`。分割不可はError。 `radiusForMass(mass:number,k:number):number`、`scaleForFrame(masses:number[], width:number,height:number):number`。

- [ ] 次を失敗させてから、1.1の式を実装する。

```ts
import {expect,test} from 'vitest';
import {splitMass} from '../../src/symbols/split-mass';
test('conserves a matching total and reaches raw values with a mismatch',()=>{
  const equal = splitMass(100,[25,75],0.5);
  expect(equal.parent + equal.children.reduce((a,b)=>a+b,0)).toBeCloseTo(100,10);
  const changed = splitMass(120,[25,75],1);
  expect(changed.parent).toBe(0);
  expect(changed.children).toEqual([25,75]);
  const middle = splitMass(120,[25,75],0.5);
  expect(middle.parent + middle.children.reduce((a,b)=>a+b,0)).toBeCloseTo(110,10);
  expect(()=>splitMass(100,[0,0],0.5)).toThrow();
});
```

- [ ] 共通縮尺は全描画円に一つだけ適用する。初期目標最大半径は `min(112, min(width,height)*0.16)` CSS px。表示集合の変更で係数が跳ねないよう、画面外マージン内の円にも距離に応じた連続重みを与え、係数を120msで補間する。係数だけの補間で分割率tは遅らせない。厳密な最大半径は静止時の目標として計測する。
- [ ] 円ごとの最大半径clampは使わない。mass=0の円は削除し、正の半径2px未満だけ描画時にロケータードットへ置換する。分割開始で点が突然現れないよう、点の不透明度は比例半径/2で0から連続的に増やす。
- [ ] 合計一致・不一致・ゼロ・負数・NaN・tの0/1境界を追加する。固定seedで100組以上の正数配列を生成し、合計がB(t)、子の比が維持されることを検証する。t=0.999と1の差が連続であることも確認する。
- [ ] `npm test -- tests/unit/split-mass.test.ts tests/unit/radius-scale.test.ts`とtypecheckを通し、`feat: add continuous population mass interpolation`をコミットする。

### Task 5: データ読込・版管理・再試行

**Files:** P/src/data/data-loader.ts、data-cache.ts、P/tests/unit/data-loader.test.ts。

**Interfaces:** `DataLoader(base:URL, fetcher:typeof fetch)`、`loadManifest():Promise<Manifest>`、`loadChildren(id:GeoId):Promise<ChildBundle>`、`loadObservations():Promise<Observation[]>`、`clearFailed(id:GeoId):void`、`dispose():void`。

- [ ] 同一親の並行要求は同じPromiseを返し、2回以上の実通信をしないテストを書く。失敗Promiseは永続キャッシュに残さない。
- [ ] 5秒timeout、失敗後500ms待機、1回再試行、合計2回失敗後に再読込可能、dispose後は応答を反映しないテストをfake timersで作る。

```ts
import {expect,test,vi} from 'vitest';
import {DataLoader} from '../../src/data/data-loader';
test('initial manifest failure is retried once',async()=>{
  vi.useFakeTimers();
  try {
    const fetcher = vi.fn().mockRejectedValue(new Error('offline'));
    const loader = new DataLoader(new URL('https://example.test/map/'),fetcher);
    const failure = expect(loader.loadManifest()).rejects.toThrow('offline');
    await vi.runAllTimersAsync();
    await failure;
    expect(fetcher).toHaveBeenCalledTimes(2);
    loader.dispose();
  } finally {vi.useRealTimers();}
});
```

- [ ] manifestの論理ファイル名から同一origin・アプリ配下のパスだけを解決する。URLにはSHAをqueryとして付ける。manifest自体はno-cacheで取得する。版が違えば別キャッシュを使用し、旧版と混ぜない。
- [ ] 読込後にスキーマ・親ID・子ID集合を検証する。通信失敗とデータ不正を区別し、不正データは自動再試行しない。
- [ ] `npm test -- tests/unit/data-loader.test.ts`を通す。応答が戻るだけで自動展開する処理はこのクラスへ置かず、Task 6が現在のカメラ状態から判断する。検証後コミットする。

### Task 6: 地域別の階層制御と可逆な進行率

**Files:** P/src/hierarchy/detail-score.ts、transition-state.ts、hierarchy-controller.ts、P/src/map/viewport.ts、P/tests/unit/hierarchy.test.ts、viewport.test.ts。

**Interfaces:** `unwrapLng(lng:number,around:number):number`、`progress(score:number,start:number,end:number):number`、`buildFrame(input:HierarchyInput):Frame`。HierarchyInputをhierarchy-controller.tsで定義し、nodes・observations・readyParents・project(node):{anchorPx,boundsPx}・viewport・前フレーム状態を受け取る。

- [ ] 最短経度差と、同じscoreなら往復とも同じtになるテストを書く。

```ts
import {expect,test} from 'vitest';
import {unwrapLng} from '../../src/map/viewport';
import {progress} from '../../src/hierarchy/transition-state';
test('reversible zoom and dateline interpolation',()=>{
  expect(unwrapLng(-179,179)).toBe(181);
  const scores = [0.2,0.5,0.8,0.5,0.2];
  const ts = scores.map(s=>progress(s,0.2,0.8));
  expect(ts[1]).toBeCloseTo(ts[3],10);
  expect(ts[0]).toBe(ts[4]);
});
```

- [ ] progressはclamp後smoothstep。ヒステリシスは読込の開始・保持だけへ適用し、描画tの分岐には使わない。ズームを戻した瞬間の不連続を防ぐ。
- [ ] scoreの寸法は投影boundsの対角長/画面対角長。画面中央60%の矩形との交差量も使用し、遠方の大きな領域が無条件に詳細化されることを避ける。主要部分と遠方飛地のboundsを分け、代表点が画面外でも領域中央に視線があれば詳細化する。
- [ ] 初期しきい値を設定化する。continent:0.75→1.25、subregion:0.65→1.10、country:0.60→1.00、admin1:0.55→0.95。ホームfitの状態では六大州表示に固定し、初期からアジアだけ分割されないようにする。変更値と理由は受入レポートへ残す。
- [ ] readyでない親のtは0。子bundleが到着して要求tが既に1なら、読み込み直後の表示だけ120ms以内で要求tへ追いつかせる。拡大操作中の通常tはズーム直結にする。motion低減時は移動させない。
- [ ] 親がt=1に達する前は子の分割を止める。階層を深く開いた状態から急縮小した場合は子孫を統合してから親の分割を戻し、二重描画を作らない。安定状態の描画集合に祖先と子孫が共存しないことをテストする。
- [ ] 世界→アジア→東アジア→日本→大阪府の合成木で、ready欠損、単一子、t往復、画面外枝、最深階層を検証する。日本のbundle到着時にカメラが欧州へ移っていれば日本を展開しない。
- [ ] `npm test -- tests/unit/hierarchy.test.ts tests/unit/viewport.test.ts`、typecheckを通してコミットする。

### Task 7: 地図描画と人口円の接続

**Files:** P/index.html、P/src/main.ts、P/src/app/bootstrap.ts、P/src/map/map-controller.ts、boundary-layer.ts、P/src/symbols/proportional-circle-layer.ts、leader-lines.ts、P/playwright.config.ts、P/tests/e2e/map.spec.ts。

**Interfaces:** `createMap(container:HTMLElement,onMove:()=>void):MapAdapter`、MapAdapter={project,unproject,getState,setState,fitWorld,resize,destroy}。`renderFrame(frame:Frame):void`、`setBoundarySelection(id:GeoId|null):void`。

- [ ] 地図が起動し六大州が描画されるブラウザテストを作る。UIの存在だけでなく、E2E専用buildで得るrender snapshotとスクリーンショットを照合する。

```ts
import {test,expect} from '@playwright/test';
test('world view starts with six population regions',async({page})=>{
  await page.goto('/');
  await expect(page.locator('[data-testid="map-ready"]')).toHaveAttribute('data-state','ready');
  const snapshot = await page.evaluate(()=>window.__populationTest.snapshot());
  expect(snapshot.stableContinentCount).toBe(6);
  expect(snapshot.hasFiniteCoordinates).toBe(true);
  await expect(page).toHaveScreenshot('world.png');
});
```

- [ ] テストAPIの型をP/src/app/test-api.d.tsに定義する。snapshotはstableContinentCount、hasFiniteCoordinates、activeIds、frameを返す。E2E専用環境変数でのみ公開し、productionではwindowへ露出させない。実画面操作をテストAPIで置き換えない。
- [ ] MapLibreは空style＋海陸塗り＋境界線。初期は2D Mercator、pitch=0、bearing=0、回転無効、world copy無効。島や日付変更線側の描画はカメラに最も近い経度へunwrapする。
- [ ] deck.glはMapLibreと同期したoverlayで開始する。ピクセル半径・billboardを使用して緯度で円面積が変わらないようにする。CPUで計算したtを各フレームで渡し、通常の分割にdeck.glの時間transitionを重ねない。
- [ ] adapterが描画位置の経緯度変換を担当する。screen-space衝突offsetをunprojectして円・引出線を同じ位置へ描画する。requestAnimationFrameで1フレーム1更新にまとめる。
- [ ] style再作成やWebGL context lostを扱い、復旧1回の後も失敗なら画面に短い案内を表示する。隠しiframeの0×0サイズでは初期fitを延期する。destroyでリスナ・GPU資源を解放する。
- [ ] Playwright chromiumとwebkitのブラウザを準備し、`npm run build`、`npm run test:e2e -- tests/e2e/map.spec.ts`で確認する。OS依存でWebGLを使えない環境は未検証として記録し、成功扱いしない。接続コードをコミットする。

### Task 8: 重なり回避・引出線・選択

**Files:** P/src/symbols/collision-layout.ts、hit-test.ts、P/tests/unit/collision.test.ts、P/tests/e2e/selection.spec.ts。Modify: renderFrameとmap選択イベント。

**Interfaces:** `layoutSymbols(symbols:PointSymbol[],maxOffset:number):PointSymbol[]`、`pickSymbol(symbols:PointSymbol[],point:XY):GeoId|null`。

- [ ] 同じ中心にある小円と大円、44px操作領域だけ重なる2円、3円以上の密集、選択中の円の分割を試験する。
- [ ] 安定ID＋大きさ順で決定的に配置する。小円の中心が大円の内部に埋もれる場合、または選択可能領域がほぼ失われる場合を回避対象とする。部分重なりだけで全円を押し出さない。
- [ ] 候補位置は固定角度・距離の順で探索し、最大offset32pxとする。大円の外へ出すのに32pxを超える場合は元位置近辺を維持し、小円を前面へ描く。完全無重複を保証しない。
- [ ] 同じ親子の分割途中は意図した重なりとして除外する。t=0.8→1で子の回避量を徐々に有効化する。32px内の解がない場合は輪郭強調と選択を優先する。
- [ ] 引出線は8px超で表示。カメラ操作中は最大10Hzで候補を更新、間を補間し、停止後に安定配置へ確定する。ラベルの当たり判定は円の判定に混ぜない。
- [ ] pickは見えている円の内部を優先し、複数なら小円、次に中心距離、最後にIDで決める。円外の透明領域では中心距離を優先する。単純な44px透明円の最前面判定に依存しない。

```ts
import {expect,test} from 'vitest';
import {layoutSymbols} from '../../src/symbols/collision-layout';
test('collision displacement stays within its declared bound',()=>{
  const symbols = [100,80,20].map((mass,i)=>({
    id:String(i),parentId:null,anchorPx:[100,100] as const,centerPx:[100,100] as const,
    rawValue:mass,mass,radiusPx:Math.sqrt(mass),locator:false,phase:'stable' as const,t:1
  }));
  const a = layoutSymbols(symbols,32);
  expect(a).toEqual(layoutSymbols(symbols,32));
  for (const s of a)
    expect(Math.hypot(s.centerPx[0]-100,s.centerPx[1]-100)).toBeLessThanOrEqual(32.001);
});
```

- [ ] unitとselectionのE2Eを実行し、東アジア・欧州・大阪市周辺で視認確認してコミットする。

### Task 9: 最小UI・URL復元・埋め込み

**Files:** P/src/app/url-state.ts、embed-mode.ts、P/src/ui/controls.ts、selection-card.ts、labels.ts、legend.ts、format-population.ts、P/src/style.css、P/tests/unit/url-state.test.ts、format-population.test.ts、P/tests/e2e/embed.spec.ts。

**Interfaces:** `parseState(url:URL):MapState|null`、`writeState(url:URL,state:MapState):URL`、`formatPopulation(value:number,compact:boolean):string`、`renderSelection(node:GeoNode|null,value:number|null):void`。

- [ ] 正常URLのround trip、不正数値、未知metric、未知ID、XSS文字列、緯度上限をテストする。不正位置は世界表示へ戻し、未知selectedIdだけなら選択を解除する。

```ts
import {expect,test} from 'vitest';
import {parseState,writeState} from '../../src/app/url-state';
test('preserves state and rejects a non-finite coordinate',()=>{
  const state = {lng:135.5,lat:34.6,zoom:8,selectedId:'jp-pref:27',metricId:'population' as const};
  const u = writeState(new URL('https://example.test/population-map/'),state);
  expect(parseState(u)).toEqual(state);
  u.searchParams.set('lat','NaN');
  expect(parseState(u)).toBeNull();
});
```

- [ ] 5秒で発見できる拡大・縮小・世界へ戻る・外部で開くを置く。選択カードは地域名・人口のみ。表示名はtextContentへ設定する。
- [ ] 面積凡例を現在の共通縮尺で描く。ロケータードットの説明は折りたたみ内へ置く。推計値の桁を勝手に増やさず原データ精度を保ち、compact表示は万・億、通常は桁区切りを使う。
- [ ] 地図のgesture領域だけtouch-actionを調整し、ページ全体のスクロールを禁止しない。埋め込みでは未操作時にスクロールを奪わず、地図をタップすると操作可能、外側タップで解除する。端末に応じ「地図を操作」を小さく表示する。
- [ ] mapのmoveendでURLをreplaceStateし、ズーム1フレームごとの履歴を作らない。URLの日本の自治体IDは都道府県コードから必要経路を読み込む。他国IDはcountry-indexから解決する。
- [ ] 外部リンクは同じURL状態でtarget=_blank、rel=noopener。window.top.locationへ書き込まない。ResizeObserverで幅高さ変化を処理する。
- [ ] 矢印移動、+/−拡縮、Home世界、Escape解除を地図にfocusがあるときだけ処理する。UIは44px以上、選択カードをaria-liveへ。motion低減は移動を省き、短いopacity切替にし、通常遷移の面積保存試験から分離する。
- [ ] 1024×768、768×1024、iframe640×480、幅375pxをE2Eで検証する。スクロール、外部リンク、URL復元、カード、motion低減を確認しコミットする。

### Task 10: 全体統合・通信例外・視覚回帰

**Files:** P/tests/e2e/journey.spec.ts、loading.spec.ts、visual.spec.ts、P/tests/e2e/fixtures/、P/tests/e2e/snapshots/。Modify: bootstrap.ts、階層・読込接続。

**Interfaces:** 既存のbuildFrame、DataLoader、renderFrame、MapAdapterを接続する。新しい独立した状態管理基盤は追加しない。

- [ ] 実データを使い、世界→東アジア→日本→大阪府→市区町村→逆方向を拡縮ボタン・wheel・dragで通す。日本全国の子bundleはデータ検証で照合し、代表UI経路とは別に全県ID読込を機械検証する。
- [ ] Playwright routeで大阪府のbundleを遅延させ、親円が残ることを確かめる。遅延中に欧州へ移動して応答を解放し、古い日本の展開が起きないことを検証する。
- [ ] 404、timeout、壊れたJSON、境界のみ欠損、再試行成功、再読込操作をそれぞれ検証する。データ欠損と人口0を同じ表示にしない。
- [ ] 世界・東アジア・日本・大阪府の静止状態、分割25/50/75%で基準画像を作る。画像は人間が見て確認してから保存し、差分が出たという理由だけで更新しない。テストfixtureと実データの表示を混ぜない。
- [ ] 地図移動中にrender snapshotから比例半径・massを取得し、同一合計で1%以内の保存、不一致合計でB(t)への一致、t=1で元人口への一致を検証する。移動中のlabel・円消失は画像でも確認する。
- [ ] WebGL停止・復旧はブラウザで可能ならWEBGL_lose_contextにより試験し、対応しない環境では手順と未実施を記録する。
- [ ] 全unit、data validation、chromium/webkit E2Eを実行し、修正差分と基準画像をコミットする。

### Task 11: 性能・データ更新・運用記録

**Files:** P/scripts/check-size.mjs、P/tests/e2e/performance.spec.ts、P/README.md、docs/superpowers/reports/population-map-acceptance.md。

**Interfaces:** check-sizeはdist内の初回取得asset＋初期dataをgzip集計してJSONを出力する。performance.specは初期地図ready時刻と10秒操作のframe時間を計測する。

- [ ] gzip集計が既知のBufferに対して `gzipSync` のbyteLengthと一致するテストを作る。全依存を含め、遅延読込を初期読込として誤集計しない。HTTP実転送量とは別欄で報告する。
- [ ] 初期通信10Mbps・RTT100ms・cold cacheの試験条件を固定する。地図描画readyまで3秒、操作中60fps目標、2秒移動窓で30fps未満が続かないことを計測する。実機とCIの結果を分ける。
- [ ] MapLibre＋deck.glを含めた1MB目標に届かない場合、tree-shaking、境界簡略化、段階読込、非初期indexの遅延化を先に試す。都合よくライブラリbytesを除外しない。
- [ ] 5秒以上の操作後・読込後・iframeリサイズ後のlong task、メモリ、フレーム落ちを記録する。必要ならWeb Workerへ衝突計算を移すが、計測で負荷が確認された処理だけを対象にする。
- [ ] 実際のiPadとNotion埋め込みの確認手順をREADMEへ記す。実機アクセスがない場合、WebKit E2E合格は実機合格と呼ばず、該当欄は「実機未測定」とする。
- [ ] 次の更新手順をREADMEに記し、source-lockのコピーでdry-runを行う。

```bash
npm run data:acquire
npm run data:build
npm run data:validate
npm test
npm run build
npm run test:e2e
npm run check:size
```

- [ ] 受入14項目の結果、端末・ブラウザ版、取得snapshot版、既知の欠損、性能目標との差をレポートに記載する。重大未達があれば試作版と明示し、正式完成と言わない。レポートをコミットする。

### Task 12: GitHub Pages用成果物と最終引き継ぎ

**Files:** P/scripts/stage-pages.mjs、P/tests/unit/stage-pages.test.ts、.github/workflows/population-map-check.yml、population-map/。Modify: P/vite.config.ts、README、受入レポート。

**Interfaces:** `stagePages(sourceDir:string, repoRoot:string):Promise<void>` は出力先 `repoRoot/population-map` のみ更新する。sourceDirはP/distのみを許す。symlinkとルート外への解決を拒否する。

- [ ] stage-pages.testで、一時repoにindex.htmlと既存教材を置き、新ビルドのcopy後もSHAが同じことを検証する。空source、symlink、出力先誤指定は失敗終了する。
- [ ] Viteのbaseは `./` とし、asset・data参照にorigin直下の絶対パスを埋め込まない。npm buildの出力先はP/dist。stage時の削除範囲はpopulation-map内の前版生成物だけとする。
- [ ] CIはpath filterでPと関連仕様・workflowのみを対象とする。npm ci→typecheck→unit→data validate→build→E2Eを実行する。生成済みデータを使い、PR検証で外部統計を再取得しない。workflowはread-only権限、Actionsは実際のrelease SHAを確認して固定する。
- [ ] 既存Pagesがブランチroot配信なら、population-map内のbuild成果物を通常の変更としてコミットする。既存Actions配信ならそのartifactへ同フォルダを追加する。設定が確認できない場合は公開設定を勝手に上書きせず、成果物とPRまで整える。
- [ ] プロジェクト名を含むサブパス `/uk-sec-visual-learning-public/population-map/` でローカルHTTP確認を行う。index、JS、CSS、各dataの404がないこと、直リンク再読込が動くことを検証する。
- [ ] 以下を実行し、保護ルールに従ってbranch pushまたは既存の統合手順を使う。変更を限定してstageし、git add .は使用しない。

```bash
npm run build
npm run pages:stage
git diff --check
git diff --stat
git status --short
```

- [ ] 公開処理を行った場合だけ実URLを取得し、HTTP 200、本文、JS/CSS/data読込、ズーム・選択を確認する。GitHubへpush成功した事実と、Pagesで動作した事実を分けて報告する。
- [ ] 最終報告に公開URLまたはPR URL、対象commit、データ基準日、実行した検証、実機未確認点、将来拡張の境界を記載する。以前のlocalhost URLは使用しない。


### 実行コマンドと設定の補足

コマンドは特記がなければ `apps/population-map/` で実行する。Taskごとの最終コミットは `git add` に当該Files欄の実パスだけを列挙して行う。

Task 1の依存導入例（導入時に公式対応範囲を確認し、導入後のpackage-lockを正とする）:

```bash
npm install --save-exact maplibre-gl @deck.gl/core @deck.gl/layers @deck.gl/mapbox
npm install --save-dev --save-exact typescript vite vitest @playwright/test @types/node
python -m venv .venv
.venv/bin/python -m pip install pandas openpyxl geopandas shapely pyproj pytest
.venv/bin/python -m pip freeze > requirements.lock
```

Python環境をactivateするか、npm scriptsのpythonを.venv/bin/pythonへ置換して揃える。NodeとPythonの実際の版をREADMEへ記録する。現在のdeck.glが専用MapLibre adapterを推奨する場合は、その公式移行案内と同一版の型定義を確認してadapterファイル内だけを変更し、選定理由を記録する。対応しないpackageをforceでインストールしない。

Task 1のVite設定の基本形:

```ts
import {defineConfig} from 'vite';
export default defineConfig({
  base: './',
  build: {outDir: 'dist', emptyOutDir: true, target: 'safari16'},
  server: {port: 5173, strictPort: true},
  preview: {port: 4173, strictPort: true}
});
```

Task 7で定義するPlaywright構成例:

```ts
import {defineConfig, devices} from '@playwright/test';
export default defineConfig({
  testDir: './tests/e2e',
  use: {baseURL:'http://127.0.0.1:4173', trace:'retain-on-failure'},
  projects: [
    {name:'chromium', use:{...devices['Desktop Chrome'],viewport:{width:1024,height:768}}},
    {name:'webkit', use:{...devices['Desktop Safari'],viewport:{width:1024,height:768}}}
  ],
  webServer:{
    command:'npm run build:e2e && npm run preview',
    url:'http://127.0.0.1:4173',
    reuseExistingServer:false,
    timeout:120000
  }
});
```

Task 7で `build:e2e` を `npm run data:validate && npm run typecheck && vite build --mode e2e` として追加する。test-apiは `import.meta.env.MODE === 'e2e'` の分岐内だけで登録する。Task 12のproduction buildでAPIが消えていることを検証する。テストはじめての際に `npx playwright install chromium webkit` を実行する。

Task 4の縮尺用関数名は `scaleForFrame`、Task 6の進行率は `progress`、Task 8の判定は `pickSymbol` に統一する。追加のキャッシュ内部型・adapter型は所有ファイルからexportし、同じ名前の類似型を別Taskで作らない。

Task 3で境界の欠損と人口の欠損を分ける。人口値がすべて揃えば境界なしでもbundleを受理するため、bounds・anchorは別の検証済み地理原本から取得してcrosswalkへ保存する。境界欠損を理由に人口エンティティを削除しない。

Task 6で「一経路一遷移」を実装する際は、要求される最終詳細度と現在描画中の詳細度を別stateにする。高速ズーム時の追従は各段120msを上限とし、逆入力で待ちキューを破棄して現在の描画状態から戻る。タッチを止めても長い分割アニメーションが続く実装は避ける。Task 10では高速度の往復入力を別ケースとして試す。

Task 9の選択中地域が分割されたときは、その地域のカードを残し境界を強調する。統合によって選択地域が表示階層の外になった場合は、表示されている直近祖先へ選択を移す。URLをそのIDへ更新する。タップ可能な極小円が完全に重なり単独選択できない場合、同地点の連続タップで候補を同じ順序で循環する。通常のカード情報量は増やさない。


## 5. 仕様との対応と完了の判断

| 原仕様の節・受入番号 | 対応Task |
|---|---|
| 目的・優先順位・初期スコープ（1〜3） | 全Task、特に1・9・12 |
| 地理階層・意味的ズーム（4〜5）、受入1〜3 | 2・3・6・10 |
| 面積・分割・補正（6）、受入4〜6 | 4・6・7・10 |
| 重なり（7）、受入7 | 8・10 |
| UI・URL（8）、受入8・10・11 | 9・10 |
| 技術・型・配信（9〜11）、受入12 | 1・3・5・7・12 |
| 出典・検証（12〜13）、受入14 | 2・3・11 |
| 読込・障害（14〜15）、受入9 | 5・6・7・10 |
| アクセシビリティ（16） | 8・9・10 |
| 性能（17） | 11 |
| テスト（18）、受入13 | 各Task・10・12 |
| 将来拡張（20〜21） | 共有型、1・3・9。追加機能は今回実装しない |

中間到達点:
- Task 3: 出典と地理IDが追跡できる全対象データ
- Task 6: 描画に依存しない連続分割と通信制御
- Task 10: 世界〜日本全国の自由探索アプリ
- Task 12: 既存教材と共存する公開成果物・検証記録

現在の計画作成では、実人口データ一式のダウンロード、Node依存固定、ブラウザ試験、Pages設定確認を実施していない。これらは上記Taskの具体的作業に含まれる。サンプルの大阪府人口や2025年の日付を検証済みデータとして流用しない。

## 6. 公式資料と確認状況

- [Vite導入](https://vite.dev/guide/)：2026-09-19閲覧。Nodeの対応条件、静的buildとbaseの確認先。
- [deck.glとMapLibre](https://deck.gl/docs/developer-guide/base-maps/using-with-maplibre)：同日閲覧。adapterとoverlayの確認先。
- [ScatterplotLayer](https://deck.gl/docs/api-reference/layers/scatterplot-layer)：半径単位・billboard・色の確認先。実装時に固定版の型定義と照合する。
- [UN WPP](https://population.un.org/wpp/)：入口は確認。動的ページ本文・最新配布ファイルは今回未取得。Task 2で解決後URLと版を記録する。
- [国連M49](https://unstats.un.org/unsd/methodology/m49/)：区分の基準。今回の再取得はtimeout。Task 2で配布表を取得し固定する。
- [人口推計](https://www.stat.go.jp/data/jinsui/)：同日閲覧。都道府県の最新公開表の入口。
- [e-Stat統計GIS](https://www.e-stat.go.jp/gis)：同日閲覧。境界のダウンロード入口。
- [e-Stat統計検索](https://www.e-stat.go.jp/stat-search?page=1&toukei=00200241)：住民基本台帳調査の取得入口としてTask 2で確認する。
- [Natural Earth利用条件](https://www.naturalearthdata.com/about/terms-of-use/)：同日閲覧。配布データはpublic domainと明記されている。

閲覧した入口ページだけで実配布ファイルを検証済みとしない。人口年、利用条件、実ファイルURLはsource-lockの取得記録を正とする。

## 7. Codexへ貼り付ける実行指示

次の文章を、そのまま実行担当のCodexへ渡せる。

> fact0404/uk-sec-visual-learning-public に、階層型の世界人口地図を実装してください。最初に docs/superpowers/specs/2026-09-13-population-map-design.md と docs/superpowers/plans/2026-09-19-population-map-implementation.md を読んでください。計画の仕様補足は、既存仕様の矛盾を解消する実装判断として優先してください。既存リポジトリとAGENTS.mdを確認し、Task 1から12まで順番に実行してください。初期版は世界の国・地域と日本全国の都道府県・市区町村の人口だけを扱います。実装はapps/population-map、公開成果物はpopulation-mapへ配置してください。定型的な選択は最善案を選び、判断理由を後で報告してください。実データ、単体テスト、地図操作、GitHub Pagesのサブパスを検証し、可能な範囲でコミット・push・公開確認まで進めてください。未測定の実機性能、取得できないデータ、権限などの阻害要因は明示し、未実施を成功と報告しないでください。

引き継ぎ時に本計画と仕様書の両方を読む。中断時は現在のTask・直近コマンド結果・残作業をREADMEまたは受入レポートへ記録し、チェック済みTaskを再実装しない。
