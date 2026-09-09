# キャラクター相関図作成ツール

[![GitHub Pages](https://img.shields.io/static/v1?label=GitHub+Pages&message=+&color=brightgreen&logo=github)](https://vvylw.github.io/character-relationship-map)
[![Deploy to GitHub Pages](https://github.com/VvyLw/character-relationship-map/actions/workflows/deploy-to-pages.yml/badge.svg)](https://github.com/VvyLw/character-relationship-map/actions/workflows/deploy-to-pages.yml)

キャラクター画像をアップロードし、円形に切り抜いてノードを作り、矢印（Relationship）でつないでキャラクター同士の相関図を作成できるWebアプリです。単一のHTMLファイルのみで動作し、サーバーやビルド環境は不要です。

対応ブラウザ：最新の Chrome / Edge / Safari / Firefox（デスクトップ・モバイル両対応）。

## 主な機能

### キャラクター作成
- 画像をドラッグ＆ドロップ、または左下の「＋」ボタンから追加（PNG / JPEG / WebP、複数枚まとめて可）。
- 複数枚を一度に追加した場合は1枚ずつ順番に切り抜き作業を行う（進捗表示付き）。
- 切り抜きは正方形ではなく円形。円をドラッグで移動、ハンドルでサイズ変更でき、選択範囲外は暗転表示される。
- 「決定」でキャラクターとして登録、「キャンセル」でその画像だけスキップ（他の画像には影響しない）。

### キャラクターの編集
- ノードをドラッグしてキャンバス上を自由に移動。
- 選択すると画面下部にインスペクターが表示され、名前の入力とカラーの変更ができる。
- カラーは Unity / lilToon 風のカスタムHSVピッカー（色相バー＋彩度・明度の2D選択エリア＋HEX/RGB入力）で変更可能。ドラッグ中はリアルタイムにプレビューされ、「OK」で確定、「キャンセル」で変更前の色に戻る。
- ヘッダーの「名前」トグルで、全キャラクターの名前表示・非表示を一括切り替えできる。
- ノードを削除すると、そのキャラクターに関わる矢印もまとめて削除される（Delete/Backspaceキー、またはノード上の「×」ボタン、インスペクターの「削除」ボタン）。

### 相関図（矢印・Relationship）
- キャラクターのハンドル（円周上の丸いアイコン）を別のキャラクターへドラッグして矢印を作成。
- 矢印は必ず一方向（source → target）。A→BとB→Aは独立したRelationshipとして扱われ、同じ向きの矢印は1組のキャラクター間に1本まで（既に存在する場合は新規作成せず、既存の矢印の説明文編集ダイアログが開く）。
- 矢印の色は固定値ではなく、**source側キャラクターの現在のカラーを都度参照**して描画される。キャラクターの色を変えると、そのキャラクターが起点になっている矢印は自動的に色が変わる。
- 矢印をクリックで選択（削除ボタン表示、Delete/Backspaceキーでも削除可）、ダブルクリックまたはラベル部分のクリックで説明文を編集。
- A→B・B→Aの両方が存在する場合、2本の矢印は湾曲させず、**中心線に対して垂直方向へ平行にオフセット**して重なりを回避する（`RELATIONSHIP_LINE_OFFSET`定数で調整可能）。
- ラベル（説明文）も対応する矢印と同じオフセット方向・符号を用いて配置されるため、矢印とラベルの対応がズレない。ラベル同士が近すぎる場合は追加で位置をずらす簡易的な衝突回避も行う。
- ラベルは常に画面内に収まるよう自動でクランプされる。

### キャンバス操作
- パン（背景ドラッグ）、ズーム（マウスホイール、または右下のズームボタン）に対応。
- 大量のキャラクターを配置しても、全体を見渡しながら編集できる。

### 保存・書き出し
- キャラクター（画像・位置・サイズ・色・名前）、Relationship（source/target/label）、表示設定（名前表示ON/OFF）、キャンバスの表示位置・ズームをブラウザのLocalStorageに自動保存し、リロード後も復元される。
- ヘッダーの「全消去」ボタンで、確認モーダルを経てすべてのデータを削除できる（誤操作防止のため、確認せずに即消去されることはない）。
- 「PNGとして保存」ボタンで、相関図全体（表示中の範囲だけでなく全キャラクター・全矢印・全ラベルを含む）を `relationships_yyyyMMddHHmmss.png`（例：`relationships_20260909153045.png`）という名前で画像として書き出せる。日時はローカル時刻を基準とする。

## データ構造（概要）

```ts
type Character = {
  id: string;
  image: string;   // 円形に切り抜かれたPNGのdataURL
  x: number;
  y: number;
  size: number;    // 表示直径(px)
  color: string;   // HEXカラー。矢印の色として動的に参照される
  name: string;    // 空文字可
};

type Relationship = {
  id: string;
  source: string;  // Character.id
  target: string;  // Character.id
  label: string;   // 空文字可
  // 色は保持しない。描画時に source キャラクターの color を参照する
};

type ChartSettings = {
  showCharacterNames: boolean;
};
```

- `source` + `target` の組み合わせは常に一意（同じ向きの矢印は1本だけ）。
- A→BとB→Aは別のRelationshipとして完全に独立しており、片方を削除してももう片方には影響しない。
- 削除は必ず `Relationship.id` / `Character.id` を基準に行う。

## 技術構成

- 依存ライブラリなしのバニラ JavaScript（ES5相当の記法）＋ SVG ＋ Canvas 2D。
- ノード・矢印描画はSVG、円形切り抜きとPNG書き出しはCanvas 2D APIを使用。
- HSVカラーピッカーもゼロから実装した自前コンポーネント（Hue/Saturation/Value個別調整、2D領域でのドラッグ選択、HEX/RGB相互変換に対応）。

## 既知の制限

- 3人以上のキャラクターが関わる複雑な矢印配置（例：A→B, A→C, B→C, C→A, C→B が偶然一直線に並ぶ場合など）では、ペア単位のオフセットだけでは重なりを完全には解消できないことがある。
- ラベルの衝突回避は簡易的なもので、非常に多数の矢印が密集した場合は完全な自動レイアウトにはならない。
- 画像・相関図データはブラウザのLocalStorageに保存されるため、ブラウザを変える／シークレットモードを使う／ストレージを消去すると復元できない。
