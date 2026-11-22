# Mini Tokyo Drive — 引き継ぎメモ

東京の3D地図の上をミニクーパー風の車で走る、ブラウザ向けドライビングシミュレータ。
成果物は単一の HTML ファイル `mini-tokyo-drive.html`(このメモと一緒に渡すこと)。

---

## 1. ここまでの経緯

| 段階 | 内容 | 結果 |
|---|---|---|
| v1 | Mini Tokyo 3D(`mt3d.Map`, npm `mini-tokyo-3d`)をラップし、`type: 'three'` レイヤーで車を追加 | **ユーザー環境で「読み込み中」のまま3分以上停止** → 不採用 |
| v2(現行) | **MapLibre GL JS 4.7.1 + OpenFreeMap(liberty スタイル)+ three.js 0.170** に載せ替え。トークン不要 | ローカルの仮地図(GeoJSON の箱ビル)で描画・走行・衝突を確認済み。**本物の OpenFreeMap 地図での動作は未確認**(作業環境から外部タイルに接続できなかったため) |

### Mini Tokyo 3D が止まった原因(推測・未確認)
- Mini Tokyo 3D は地図タイルの他に、電車データ(`https://minitokyo3d.com/data`)・スタイル・辞書を取得し、1つでも失敗すると `load` イベントが来ない作り。
- 候補:`file://` で開いた/他オリジンからのデータ取得が CORS で拒否/ODPT トークン未設定/Mapbox トークン無効。
- npm の最新版は `4.1.0`(2026-11 時点)。

---

## 2. 現行の構成(mini-tokyo-drive.html)

### 読み込んでいるもの(CDN)
- `maplibre-gl@4.7.1`(UMD スクリプト + CSS)— グローバル `maplibregl`
- `three@0.170.0`(importmap で `three` / `three/addons/`)— `GLTFLoader` を使用
- 地図スタイル:`https://tiles.openfreemap.org/styles/liberty`(`CONFIG.styleUrl` で変更可。`bright` / `positron` もある)
- スクリプトは `<script type="module">`

### コード構成(上から順に)
1. **CONFIG** — スタート地点、`carScale`、車モデル差し替え設定、走行性能、カメラモード
2. **SPOTS** — ワープ先 9か所(東京駅・銀座・渋谷・新宿・東京タワー・秋葉原・皇居・芝浦・浅草)
3. **ユーティリティ** — clamp / lerp / 角度補間 / toast
4. **buildMiniCar(THREE)** — 手続き生成のミニ風カー
5. **makeBlobShadow** — 車の下の丸い影(キャンバスで作った放射グラデーション)
6. **pointInRing / pointInPolygon / pointInGeometry** — 建物衝突用の点と多角形の内外判定
7. **Engine** — WebAudio のエンジン音(ノコギリ波 + 矩形波 → ローパス)、衝突音 `thud()`
8. **map 生成** — `interactive: false`(地図のドラッグ・キー操作は無効)、`maxZoom 22`, `maxPitch 85`
9. **carLayer** — MapLibre の `type: 'custom'` レイヤー。three.js の renderer を地図の GL コンテキストで共有
10. **load 処理** — 地名を日本語優先(`name:ja`)に変更、`fill-extrusion` レイヤーを建物として収集、車レイヤー追加
11. **入力・UI** — キーボード、ホイールズーム、ワープ選択、カメラ・衝突ボタン
12. **physics / updateCarObject / updateCamera / updateHUD / loop**

### 座標系(重要)
- **車の状態は実寸メートル**:`car.lng/lat`(位置)、`car.heading`(ラジアン、北=0・時計回り)、`vx, vy`(東・北方向 m/s)、`vf`(前後速度)。
- 移動量の度換算:緯度 1° = 111132.9 m、経度 1° = 111319.5 × cos(緯度) m。
- **車モデルの軸**:Y が上、前方が **-Z**、右が +X、1 単位 = 1m(three.js の標準的な向き)。
- **描画**:カスタムレイヤーの `render(gl, args)` で
  `projection = 地図の行列 × 平行移動(車の Mercator 座標) × 拡大縮小(s, -s, s) × X軸回転(π/2)`
  を毎フレーム作る(MapLibre 公式の three.js サンプルと同じ方式)。`s = meterInMercatorCoordinateUnits × carScale`。
  車の向きは `carHolder.rotation.y = -heading`。
  車を常に原点付近に置いて行列を作るので、高ズームでもガタつきにくい。
- MapLibre v5 に上げる場合、`render` の第2引数がオブジェクトになる(`args.defaultProjectionData.mainMatrix`)。コードは両対応済み。

### 見た目の拡大(carScale = 2.4)
地図カメラは最大ズーム(22)でも車から数十m離れるので、実寸の車だと小さすぎる。見た目だけ拡大し、**物理・衝突判定は実寸のまま**。

### 物理(簡易アーケード)
- 自転車モデル:ヨーレート = `vf / ホイールベース × tan(ステア角)`
- ステア角は速度が上がるほど小さくなる(低速 0.6 rad → 高速 0.09 rad)
- 速度を新しい向きで前後・横に分解し、横成分を `exp(-grip × dt)` で減衰 → 横滑りが残りドリフトになる。サイドブレーキ中は grip 11 → 1.4
- 加速は最高速に近づくほど弱まる。空気抵抗 + 転がり抵抗あり
- 高速時は 1 フレームを細かいサブステップに分けて計算(建物のすり抜け防止)
- 見た目用にロール・ピッチ(横G・加減速で車体が傾く)
- ギア・回転数は速度から作った演出用(変速点 22/45/75/110/145 km/h)

### 建物衝突
- 進行方向側のバンパー位置(車長 3.85m の半分先)を画面座標に変換し、`queryRenderedFeatures` で `fill-extrusion` レイヤーを ±3px の範囲で取得。
- 3D の建物は手前のビルも拾うので、返ってきた建物の形状(経緯度)に対して点が中にあるかを改めて判定。
- 当たったら移動を取り消し、速度 × -0.25 で跳ね返り、トースト表示・衝突音・画面揺れ。
- **制約**:画面に描かれている建物しか判定できない(画面外はすり抜け)。

### カメラ
`map.jumpTo` を毎フレーム実行。bearing は車の向きに遅れて追従(滑らかに見せるため)。`padding.top` で車を画面下寄りに配置。速度が上がるほど少し引く。

| モード | zoom | pitch | 画面上の余白 |
|---|---|---|---|
| 追従 | 20.6 | 74 | 0.42 |
| 遠景 | 19.2 | 66 | 0.30 |
| 真上 | 19.6 | 0 | 0 |
| 低空 | 21.3 | 82 | 0.50 |

### 操作
↑/W アクセル、↓/S ブレーキ・バック、←→/AD ハンドル、Space サイドブレーキ、C カメラ、ホイール ズーム、L ライト、B 衝突 ON/OFF、M 音、R リセット、H ヘルプ表示切替。
追加分:F/V 上昇・下降(空飛ぶ車)、X 乗り物、K ホーン、G ルート線、O 配達オンライン、Enter 注文を受ける、P/N 音楽。ヘルプはスタート 15 秒後に自動で隠れる。タイトル画面はクリックかキーで開始(音を鳴らすのにユーザー操作が必要なため)。

### ミュージックプレイヤー(画面右上)
- `Music`(即時関数)。`<audio>` 要素で MP3 / MP4 / M4A を再生(エンジン音の WebAudio とは独立)。
- **📁 フォルダ**:Chrome/Edge は `showDirectoryPicker`(サブフォルダも再帰で探す)。ハンドルは IndexedDB(`mini-tokyo-drive-music`)に保存し、次回起動時に許可済みなら自動で一覧を読み込む/未許可ならフォルダ名をクリックで再接続。非対応ブラウザは `<input webkitdirectory>` にフォールバック(記憶なし)。
- **`CONFIG.musicFolderUrl`**:HTTP 配信時に `'music/'` などを指定すると、サーバーのディレクトリ一覧(`python -m http.server` 等)から曲を拾う。こちらが優先。
- 曲はパスの自然順で並ぶ。シャッフル、リピート(全曲/1曲/なし)、シーク、音量(localStorage に保存)、折りたたみ、Media Session(OS のメディアキー)対応。
- キー:P 再生/一時停止、N 次の曲。スライダー操作中はキーを車に渡さず、操作後にフォーカスを外す。プレイヤー上のホイールは地図ズームにしない。

### 乗り物(VEHICLES)
- `scooter`(原付・既定)/ `bicycle` / `car`。走行性能・見た目の拡大率(`scale`)・カメラの寄り(`camZoom`)・バンク角(`lean`)・変速(`gears`、null は CVT 表示 D)・エンジン音・ホーンを1か所に定義。`physics` は `V`(今の乗り物)を読む。
- 二輪はカーブで内側に倒れる(`atan(横G / g)`)。モデルは `buildScooter` / `buildBicycle`(配達員 `buildRider` 付き。原付は荷台ボックス、自転車は四角いリュック)。自転車はクランクが回る。
- タイトル画面のカード、上部のセレクト、X キーで切替(localStorage に記憶)。K でホーン/ベル。

### 空飛ぶミニ(flycar)
- タイトル画面では「???」のシルエットのカードのみ。スタートまで 3D モデルも表示しない(`carHolder.visible`)。
- `flight.phase`:`takeoff`(自動離陸。1.2 秒地上で噴射 → `hoverAlt` 5m まで上昇。この間は操作不可)→ `fly`。高度 `car.alt`(m)、上下の速度 `car.vz`。F 上昇 / V 下降、離せばホバリング。0〜600m。
- タイヤは高度 `gearUpAlt`(2.5m)を超えると約 1.2 秒で内側に倒れながら持ち上がって収納、下回ると出る(`flight.gear`)。
- モデル `buildFlyCar` = ミニ + 下向きジェット 4 基(炎は `thrust` で伸縮・ちらつき)+ 後部ブースター 2 基(アクセルで噴く)+ 発光ライン。地面の照り返し `makeGroundGlow`、影は地面に残して高度で薄く。
- 描画は `MercatorCoordinate.fromLngLat(..., car.alt)`。カメラは、注視点を車の真下から `alt×tan(pitch)` 前へずらし、`alt/cos(pitch)` 分ズームアウトして、車を画面上の同じ位置・大きさに保つ。
- 建物の衝突は `render_height` より高ければすり抜け。ジェット音はホワイトノイズ+バンドパス。速度計の GEAR 表示は ALT(高度)になる。
- ナビは飛行中、ルートを外れても再検索しない。配達は高度 12m 未満まで降りないと受け取り・お渡しできない。

### 空のじゃま者(Hazards・空飛ぶミニで飛行中のみ / `CONFIG.flyHazards`)
- 位置は「飛行開始地点 origin からの東・北 m + 高度」で持ち、毎フレーム車からの相対位置に置く。three.js のシーン原点は車(高度 car.alt)なので y = 高度 − car.alt。`root.scale = 1/V.scale` でメートル単位のまま置ける。
- **カラス**:飛行開始 8 秒後から 5〜9 秒ごとに 1〜3 羽、前方 170〜250m に出現(最大 6 羽)。車の少し先へ旋回しつつ車の 18m 上を保つ。フンは「落下時間 √(2h/g) の間に車が今の速度のまま進む位置」と着地点が 4m 以内に重なったら落とす(1 羽 3 発まで)→ 進路や速度を変えれば避けられる。直撃で画面にフンが付く(#splat、3.5 秒で消える)。体当たりでも揺れる。
- **大怪獣ガルザウルス**(オリジナルデザイン・全高約 75m):飛行開始 `kaijuFirstDelay` 秒後、以降は去って/倒して `kaijuInterval` 秒後。`Nav.pointAhead()` でルート 350〜550m 先(ルートが無ければ進行方向)を、道と直角に 7m/s で 640m 横断。足音で地響き(距離で減衰)。
- **非常線**:怪獣の周り半径 90m に「立入禁止」テープの輪 3 本(不透明)。※ 大きな半透明の円筒にすると、MapLibre と共有している描画で真っ白/真っ赤に描かれて怪獣が見えなくなったため、半透明は使っていない。高さに関係なく入れない(`physics` で跳ね返す)。怪獣が歩いてきたら外へ押し出す。スマホの地図に赤い円、怪獣・カラスの点も表示。
- **ロケット弾**(E・長押しで 0.3 秒ごとに連射。怪獣の出現中のみ):前方 ±40°・900m 以内なら LOCK(画面の赤い円、`carLayer.lastProj` で 3D → 画面座標)し誘導。胴 −1、頭(60m 以上)−2、HP 24。半分で再び咆哮。倒すと倒れて沈み、`CONFIG.kaijuBonus`(5000 円)を `Delivery.addBonus` で売上に加算。カラスも撃ち落とせる。
- 効果音は WebAudio で合成(咆哮・ロケット・爆発・カァ・べちゃ)。
- デバッグ:コンソールで `hazards.kaiju()`(すぐ出現)、`hazards.crows()`、`hazards.state()`。
- 制約:ロケット・怪獣・カラスは建物をすり抜ける。
- 制約:飛行中はカメラが車の後ろへ大きく引くため、丸の内など超高層ビル街ではカメラがビルの中に入り、画面がビルの壁(灰色)で覆われて車や怪獣が見えなくなることがある。対策案:飛行中は建物を半透明にする/カメラ位置のビルを検出して pitch を下げる など。

### デバッグ・動画撮影用のフック
- `window.debug`:`keys`(押下状態を直接操作)、`setCollisions(bool)`、`setVehicle(id)`、`warp({lng,lat,heading})`、`navRoute()`(ナビのルート座標)、`mini`(スマホの地図)、`flight`。
- `delivery.debugOffer({name, ll}, {name, ll}, reward?, limitSec?)`:お店とお届け先を指定して注文を出す。`nav.getRoute()`。

### チュートリアル動画
- `video/mini-tokyo-drive-tutorial.mp4`(1600×900・30fps・約 2 分 41 秒)。MC は `mc_closed.png` / `mc_open.png`(ナレーションの音量で口パク)。ナレーションは Windows の日本語音声(Haruka)。
- 撮影はヘッドレス Chromium で、`performance.now` / `requestAnimationFrame` / タイマー / CSS アニメを仮想時計に置き換えて 1 コマずつ進めて撮影(描画が遅くても滑らか)。自動運転はルートを 3m 間隔に細分化して少し先の点を追う方式。

### 配達ゲーム(Delivery)・スマホ画面
- 左上はスマホ枠の独自アプリ「まちデリ」(ステータスバー・今回の売上・オンライン切替・地図・ナビ・注文シート・累計)。
- 流れ:オンライン(O)→ 注文(15秒以内に受ける/Enter)→ お店へ → 30m 以内で 8km/h 未満を 2 秒 → お届け先へ → 1.5 秒で完了。
- お店は `querySourceFeatures('openmaptiles', {sourceLayer:'poi'})` で地図データの実在の飲食店(200〜1800m)。無ければ架空の店名とランダムな地点。お届け先はお店から 500〜1800m のランダムな地点(OSRM で道路に寄せる)。
- 報酬 = 250 + 店までの道のり×0.04 + 配達距離×0.15 円。制限時間 = 道のり ÷ 乗り物の `avgKmh` + 120 秒。
- 評価:遅れ(−1、2分超 −2)、お届け中の衝突(12km/h 超)1回ごと −1(最大 −2)。遅れも衝突もなければチップ 50〜250 円。キャンセルは評価 3 として記録。
- ランク:ルーキー → ブロンズ(5件)→ シルバー(15)→ ゴールド(40)→ プラチナ(100)。累計は localStorage(`mtd-delivery-stats`)。
- 配達中のナビは `managed` モード(到着で自動終了しない・✕ を隠す・手動の目的地変更を拒否)。
- メイン画面のルート線は既定で非表示(`CONFIG.showRouteOnMain`)。G キー / 🛣 ボタンで切替(記憶される)。

### タイトルに戻る
- 上部の「🏠 タイトル」または Esc。`backToTitle()`:ループ停止(`rafId`)・キー解除・配達をオフライン(`Delivery.reset`、配達中は確認ダイアログ)・ナビ終了・`Hazards.clear()`・エンジン音停止・乗り物を地上に戻す(空飛ぶミニは次のスタートで再び離陸)。場所はそのまま。タイトル画面での Esc は何もしない。

### 目的地の入力(DestSearch)
- 上部の「🏁 目的地」は入力欄。フォーカスでおすすめ(SPOTS)を表示、文字を入れると絞り込み、Enter で検索、↑↓ で選んで Enter かクリックで決定。日本語変換中の Enter は無視。
- 住所っぽい入力(都・区・市・町・村・丁目・番・号・数字を含む)→ **国土地理院 住所検索 API**(`msearch.gsi.go.jp/address-search/AddressSearch`)を優先。地名・施設名 → **Nominatim**(OpenStreetMap)を優先し、国土地理院の結果は入力をそのまま含むものだけ。どちらもキー不要・CORS 可。
- 結果は東京都内のおおよその範囲(経度 138.9〜140.0、緯度 35.45〜35.95)に絞る。Nominatim は 1 秒 1 回までの規約なので、検索は Enter のときだけ。
- 入力した文字列はこの 2 つの外部サービスに送られる。

### ミニマップ・ナビ(画面左上)
- `Mini`:2枚目の MapLibre(`CONFIG.miniStyleUrl`、既定 OpenFreeMap bright)。`interactive: false` で、約30fps で車の位置へ `jumpTo`。「進行方向が上/北が上」切替、ホイールで拡大縮小。車アイコンは地図上ではなく CSS の重ね表示。
- `Nav`:目的地はミニマップをクリック、または上部の「🏁 目的地…」(SPOTS)で指定。経路は **OSRM**(`CONFIG.routerUrl`、既定は公開デモサーバー。軽い利用のみ可・1秒1回まで)。車の向きを `bearings` で渡し、失敗したら付けずに再試行。
- ルート線はメイン地図(建物の下・指数で太さ変化)とミニマップの両方に `route` ソースで描く。
- 5回/秒、ルート上の最寄り点を求めて残り距離・次の曲がり角・到着を判定。`CONFIG.offRouteMeters`(35m)以上外れて1.5秒経つと再検索(5秒に1回まで)。
- 案内文は OSRM の maneuver(type/modifier)から日本語を自前で生成。道なりの「名前が変わるだけ」の案内は省く。音声は `speechSynthesis`(ja-JP)で 300m 手前と「まもなく」。話している間は音楽を小さくする(`Music.duck`)。

### デバッグ用
`window.map`(MapLibre)と `window.car`(車の状態)を公開している。例:コンソールで `car.lng = 139.70; car.lat = 35.66` でワープ。

---

## 3. 車モデル(仮)と差し替え方法

**仮モデル**:赤いボディ・白いルーフ・ボンネットの白ストライプ・丸目ヘッドライト(メッキの輪)・楕円グリル・縦長テールランプ・白いドアミラー・黒いフェンダーアーチ・3本スポークのホイール。
- ボディとガラス部分は、横から見た輪郭を押し出して作っている(`extrudeProfile`)
- ホイールは `pivot`(ステア用)→ `spin`(回転用)の2段構造
- ヘッドライト・テールランプは発光の強さで点灯を表現(ブレーキ時はテールが強く光る)

**glTF/GLB への差し替え**:`CONFIG` で設定する。
```js
carModelUrl: 'models/mini.glb',
carModelRotationY: 0,        // 前方が +Z のモデルなら Math.PI
carModelScale: 1,
carModelOffsetY: 0,
carModelWheelNames: { fl: 'Wheel_FL', fr: 'Wheel_FR', rl: 'Wheel_RL', rr: 'Wheel_RR' }, // 任意(タイヤ回転用)
```
差し替え後もロール・ピッチは適用される。ライトの点灯演出は手続き生成モデルでのみ動く。

---

## 4. テスト方法(作業環境で使ったもの)

外部の地図タイルに接続できない環境でも、npm で入れたライブラリと仮の地図で検証できる。
1. `npm i maplibre-gl@4.7.1 three@0.170.0 puppeteer-core`
2. HTML 内の CDN の URL を `/node_modules/...` に置き換え、`styleUrl` をローカルの `style.json` に差し替える。この `style.json` は背景・道路の線・GeoJSON の箱ビル(`fill-extrusion`、高さは `["get","h"]`)だけのもの
3. `python3 -m http.server` で配信し、puppeteer + ヘッドレス Chrome(`--use-gl=swiftshader --enable-unsafe-swiftshader`)で、キーを押してスクリーンショットを撮る
4. 衝突の確認:`page.evaluate` で `car` をビルの手前に置き、↑を押し続けてビルの縁で止まるか見る(確認済み:縁の手前で停止)

---

## 5. 未確認・既知の課題

- [ ] **本物の OpenFreeMap 地図での動作確認**(まずこれ)。確認したいこと:
  - liberty スタイルの 3D 建物レイヤー名と、何ズームから表示されるか
  - `name:ja` が使えるか
  - 高ズームで車がガタつかないか
- [ ] 画面外の建物はすり抜ける
- [ ] 地面は平ら(標高・高架・トンネルなし)。首都高や橋の上は走れず、地上扱い
- [ ] 道路に沿う仕組みはなく、どこでも走れる
- [ ] MapLibre v4 には空(sky)の描画がない → v5 に上げれば空や霧を出せる
- [ ] 昼夜の変化なし(v1 では Mini Tokyo 3D の時間帯ライティングに追従していた)
- [ ] スマホ向けのタッチ操作なし

---

## 6. 次にやれそうなこと(候補)

1. **PLATEAU の3D都市モデル**(国交省の 3D Tiles)を重ねてリアルな街並みにする
2. **リアルタイムの電車を復活**:ODPT(公共交通オープンデータセンター)の API から列車の位置を取得し、three.js で電車を置く。または Mini Tokyo 3D のリポジトリを自分で配信して v1 方式に戻す
3. **道路への吸着・走れる場所の制限**:OpenMapTiles の `transportation` レイヤーを使う
4. **ゲーム要素**:チェックポイントのタイムアタック、ミニマップ、他の車(AI)
5. **時間帯・天気**:太陽の位置から光源を計算、夜はヘッドライトを点灯、雨のパーティクル
6. **MapLibre v5 へ移行**(空・霧・地球表示など)

---

## 7. ライセンス・クレジット

- MapLibre GL JS(BSD-3)、three.js(MIT)
- 地図:OpenFreeMap / © OpenMapTiles / © OpenStreetMap contributors(画面右下に帰属表示あり)
- Mini Tokyo 3D(MIT, © Akihiko Kusanagi)— v1 で使用。現行版では未使用
