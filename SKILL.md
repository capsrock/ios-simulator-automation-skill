---
name: ios-simulator-automation
description: Xcode 27.2 以降の iOS シミュレータをコマンドと MCP で操作・検証するときの道具の選び方と落とし穴。Device Hub を開かずに済ませる方法。タップ・文字とキーの入力・キーボードショートカット・撮影・回転・要素ツリーの取得・可変ウインドウの幅・iPhone Duo の開閉と内側へのタップ・状態の作り込み、Claude のシミュレータパネル / AXe / simctl / devicectl / mobilebuildmcp / Xcode MCP の使い分けを扱う。シミュレータで画面を確認したい、タップが効かない、要素の座標が取れない、キー入力やショートカットを試したい、回転させたい、ウインドウの幅を変えたい、スクリーンショットを撮りたい、UI 自動操作が不安定、Device Hub を開かずに済ませたい、といったときに使う。
---

# iOS シミュレータの自動操作

シミュレータを操作する道具は複数あり、**同じことができる道具でも、動くものと動かないものが混ざっている**。
何を使うかだけでなく、**なぜそれを選んだのか**を区別して持っておく。壊れているから避けているのか、
それが本来の道なのかが分からないと、下の層が直ったときに見直せない。

## 対象バージョン

**Xcode 27.2 以降、iOS 27.2 以降のシミュレータ、AXe 2.1.1 以降（Homebrew 版）。**
実測は 2026-11-21（Xcode 27.2、iOS 27.2、AXe 2.1.1。iPhone 18 Pro・iPhone Duo・iPad Pro 11-inch (M5) のシミュレータ）。

**AXe は `axe --version` で版を確かめてから使う。** これより古い版では、タップとキー入力が成功と返すのに効かない。

複数の Xcode を入れている場合、`xcrun` も MCP も **`xcode-select` が指すほう**を使う。
別バージョンのランタイムを触るときは、そのコマンドにだけ `DEVELOPER_DIR=...` を付ける。

## 最初に守ること

- **Device Hub は開かない。** このスキルの操作は、すべて Device Hub を開かずにできる。Device Hub は画面を占有してユーザーの作業を止める。
  実測は Device Hub を起動せず、`xcrun simctl boot` で起動したシミュレータで行った。
  タップ・キー入力・ショートカット・撮影・回転・`describe-ui`・可変ウインドウの幅・iPhone Duo の開閉と回転が届いた。
  ユーザーが Device Hub を開いているときは、**終了させない。** シミュレータも一緒に落ちる
- **返り値ではなく、読み直して確かめる。** 成功と返して効いていない道具がある（下の「mobilebuildmcp」）。
  `describe-ui` で値を読むか、撮って見る
- **読み出した値を報告に書く。** 「確認しました」だけでは、その確認をした証拠にならない
- **座標は `describe-ui` の `frame` から取る。** 撮った画像から目測するのは最後の手段
- 1つの道具で届かなかっただけで「届かない」と言わない。別の道具でも試す
- 数値が説明できないときは、**一時的な計測コードを入れて実測する。** 推測で定数を置かない。
  `onGeometryChange` で `proxy.size` と `proxy.safeAreaInsets` をオーバーレイに出し、確認後に必ず外す

## 道具の選び方

| やりたいこと | 使うもの | なぜ |
|---|---|---|
| タップ | Claude のシミュレータパネル `mcp__Claude_Code_iOS_Simulator__control` の `tap`、または `axe tap -x -y` | 縦・横・iPhone Duo の内側・iPad とも届く。パネルは手数が少ない |
| 要素の座標・ラベル | `axe describe-ui` | 座標を取る唯一の手段。画像から目測するより正確 |
| 要素を指定して押す・まとめて押す | mobilebuildmcp の `snapshot_ui` → `tap` / `batch`（AXe の場所の指定が要る） | 座標を使わず `elementRef` で押せる（下記） |
| 文字・キーの入力 | `axe type "<text>"` / `axe key <番号>` | 入力欄に届く（下記） |
| キーボードショートカット | ⌘ 付きは `axe key-combo --modifiers <番号> --key <番号>`、修飾キーなしは `axe key <番号>` | どちらも `keyboardShortcut` が効く（下記） |
| キーを直接受ける画面（`onKeyPress`） | `axe key <番号>` / `axe key-combo` | フォーカスのある view の `onKeyPress` に届く（下記） |
| 撮影 | パネルの `screenshot`、または `xcrun simctl io <UDID> screenshot` | どちらも撮れる。ファイルに残すなら `simctl io`。iPhone Duo では `--display` が要る（下記） |
| 回転 | `xcrun devicectl device orientation set` | `simctl` に回転が無い。Xcode 27 の正規の手段 |
| 回転＋タップ＋階層をまとめて | Xcode MCP の `DeviceInteractionSynthesize` | 1回でスクリーンショットと hitPoint 付き階層が返る |
| 外観・文字サイズ・コントラスト | `xcrun simctl ui` | 正規の手段 |
| 画面の状態づくり | `axe tap` で UI を操作する | メニューを開いて項目を選び、元に戻すまで取りこぼさなかった。ツールバーのメニューのボタンは `describe-ui` に出ないことがあり、その座標は撮影から読む |
| 状態の上書き（時刻・電波など） | `xcrun simctl status_bar` | 正規の手段 |
| 可変ウインドウの幅 | `xcrun devicectl device appResize start` / `set` | 前面のアプリの幅と高さを変えられる。iPhone だけ（下記） |
| iPhone Duo の開閉・回転・状態・撮影 | `axe hinge set` / `axe rotate` / `axe displays` / `axe screenshot` | AXe だけで足りる（下記） |

## よく使う組み立て

```bash
UDID=<simulator-udid>
AXE=/opt/homebrew/bin/axe

# 起動する（Device Hub は開かない）
xcrun simctl boot $UDID
xcrun simctl bootstatus $UDID -b

# 要素の座標を取る
$AXE describe-ui --udid $UDID

# 押す
$AXE tap -x 344 -y 255 --udid $UDID

# 打つ（入力欄にフォーカスを置いてから）
$AXE type 'abc123' --udid $UDID
$AXE key 42 --udid $UDID                              # バックスペース
$AXE key-combo --modifiers 227 --key 31 --udid $UDID  # ⌘2

# 撮る
xcrun simctl io $UDID screenshot out.png

# 回す
xcrun devicectl device orientation set --device $UDID landscapeLeft
xcrun devicectl device orientation get --device $UDID

# 状態を作る（UI を触らない）
xcrun simctl status_bar $UDID override --time 9:41 --cellularBars 4 --batteryState charging
xcrun simctl ui $UDID appearance dark
```

複数ディスプレイを持つ端末は `--display` で選ぶ。`xcrun simctl io <UDID> enumerate` で一覧が出る。

## 文字・キーの入力とショートカット

- 入力欄にフォーカスを置いてから送る
- **`axe key` と `axe key-combo --key` には、キーの文字ではなく番号を渡す。** 番号はキーそのものに付いた番号で、
  USB キーボードの規格（HID）で決まっている。文字コードとは違う（`a` は ASCII では 97、キーの番号では 4）。
  キーの番号なので `a` と `A` は同じ 4 で、大文字は Shift（225）と一緒に押す

  | キー | 番号 |
  |---|---|
  | a〜z | 4〜29（a が 4、b が 5 … の連番） |
  | 1〜9、0 | 30〜38、39 |
  | Return、Esc、Backspace、Tab、Space | 40、41、42、43、44 |
  | →、←、↓、↑ | 79、80、81、82 |

- ⌘ などは `--modifiers` に番号で渡す。Control 224、Shift 225、Option 226、Command 227。
  例：⌘2 は `axe key-combo --modifiers 227 --key 31`
- **`onKeyPress` が `.handled` を返すと、⌘ 付きも含めてキーはそこで止まり、`keyboardShortcut` は動かない。**
  ショートカットを試す画面では、`onKeyPress` が `.ignored` を返すようにする
- **キーボードが日本語入力だと、`axe type` の文字が変換されることがある。** iPad では `abc123` が `亜bc123` になった
  （最初の文字だけが変換された）。iPhone 18 Pro では変換前の文字のまま入った。英数字を確実に入れるなら、
  入れたあとに欄の値を `describe-ui` で読み直す
- `axe type` はシミュレータのキーボード配列に合わせて打つ。US の配列で送るには `--layout us`
- キーの試験は、受け取ったキーを `Text` に出す小さなアプリを作ると早い（XcodeGen でプロジェクトを作り、`simctl install` で入れる）。
  `accessibilityIdentifier` を付けておくと、`describe-ui` で値を読める

実測:

| 試したもの | 結果 |
|---|---|
| `axe type "abc123"`（Safari のアドレス欄） | 届いた。欄の値が `abc123` |
| `axe key 42`（バックスペース） | 届いた。`abc123` → `abc12` |
| `axe key-combo --modifiers 227 --key 31`（⌘2） | 届いた。SwiftUI の `.keyboardShortcut("2", modifiers: .command)` を付けたボタン（タブの切り替え）が動いた |
| `axe key 14`（k）・`axe key 44`（Space） | 届いた。`.keyboardShortcut("k", modifiers: [])` と `.keyboardShortcut(.space, modifiers: [])` を付けたボタンが動いた |
| `axe key 4`（a）・`key 14`・`key 44`・`key-combo --modifiers 227 --key 13`（⌘J） | 届いた。フォーカスのある view の `onKeyPress` が、`press.characters` で a・k・空白・j を受け取った |

## 可変ウインドウの幅

`devicectl device appResize` で、**前面のアプリ**を `Resizable` という名前のディスプレイに移し、幅と高さを変える。

1. アプリを前面に出す（`xcrun simctl launch <UDID> <bundle id>`）
2. `appResize start` でセッションを始める。**止めるまで動き続ける**ので、`-t <秒>` で時間を区切るか、Ctrl+C で止める
3. セッションのあいだに、別のコマンドで `appResize set` を打つと大きさが変わる
4. `axe describe-ui` でアプリの `frame` が変わったかを確かめる。`--display=resizable` で撮るとリサイズした画面が写る
5. セッションが終わると、元の大きさに戻る

```bash
xcrun simctl launch $UDID <bundle id>
xcrun devicectl device appResize start --device $UDID --preferred-size 600x800 -t 30 &
sleep 7
$AXE describe-ui --udid $UDID          # アプリの frame が 600x800
xcrun devicectl device appResize set --device $UDID --preferred-size 450x700
xcrun simctl io $UDID screenshot --display=resizable out.png
wait
```

- 指定できる大きさは 375×375〜1280×1280（iPhone 18 Pro）
- 撮った画像は指定した大きさの 3 倍（600×800 → 1800×2400）
- 前面にアプリが無いと `error 24001`（There is no foreground application to move to the resizable display）で弾かれる
- **iPad のシミュレータでは使えない。** `This simulator's runtime does not support resizable app windows.
  (com.apple.dt.CoreDeviceError error 1001)` で止まる（iPad Pro 11-inch (M5)、iOS 27.2）
- **iPhone Duo は閉じた状態でだけ使える。** 開いた状態では `Resizing is not available on this device right now.` で止まる。
  閉じた状態では動くが、大きさが指定どおりにならない（400×600 を頼んで 400×667）。`describe-ui` で読み直す
- `simctl io <UDID> screenshot` を `--display` なしで撮ると、メインの画面を撮るのでリサイズした画面は写らない

## iPhone Duo

**操作はすべて AXe で足りる。** 開閉・半折り・状態の読み取り・回転・撮影・内側と外側へのタップ・長押し・スワイプ・
アプリへのキー入力が届いた（2026-11-21）。

- **開閉は `axe hinge set`。** `open`（180°）・`closed`（0°）・`tabletop`（90°）・`<0〜180 の角度>`
- 開くと内側の画面（951×669、横向き）、閉じると外側の画面（466×678、縦向き）が出る
- **角度は指定どおりにならない。** `tabletop` は 94.6°、`120` は 118.5° になった。エラーは出ない。
  角度が大事なときは `axe hinge read` で読み直す
- 今出ている画面とヒンジの角度は `axe displays` か `axe hinge read` で読む。`axe displays` では画面の大きさと向きも分かる。
  `axe screenshot` は、内側・外側のうち今出ているほうを撮る
- 回転は `axe rotate left` / `right`。今出ている画面の向きを変える。
  **回したあとは 5 秒ほど待ってから向きを読む。** 2 秒では前の向きのままに読めた。
  前面のアプリが回らないと（「設定」など）`Error: The rotation did not complete.` になる
- **開閉・半折りのあとは、座標を読み直す。** アプリの並びが変わることがある。
  「設定」は半折りでサイドバーの幅が 280 から 416 に広がり、前の座標で押すと隣の行に当たった
- **撮影の `simctl io <UDID> screenshot` には `--display` が要る。** 付けないと
  `Cannot choose between 2 displays.` で撮れない。外側は `--display=primary`（1398×2034）、
  内側は `--display=primary-1`（開いた状態で 2853×2007）。**出ていないほうの画面を撮ると真っ黒になる。**
  今出ている画面を撮るなら `axe screenshot` が簡単（内側・外側とも、正しい向きで撮れた）
- **内側へのタップは、パネルの `tap` でも `axe tap` でも届く。** パネルの座標の範囲（`attach` が返す `951x669`）は
  内側の画面と同じで、`describe-ui` の `frame` をそのまま使える。長押しは `axe tap --hold 1`、スワイプは `axe swipe`
- **閉じても外側に切り替わらなくなることがある。** 角度は 0° なのに、内側が出たままになった。
  そうなると、どの道具で開閉しても戻らず、`xcrun simctl shutdown` → `boot` で直った。原因は分からない
- **`axe swipe` のあとに `Device input is quarantined` と出て、以後の入力が止まることがある。**
  送っているあいだに画面の監視が途切れ、指が離れたかを確かめられなかったときに出る（スワイプ自体は届いていた）。
  画面がおかしくないかを撮って確かめてから、エラー文が示す `*.quarantine.json` を消すと入力が戻る
- **ヒンジが動かせなくなることがある。** `axe hinge read` も `hinge set` も `Error: The hinge could not be read or set.` で止まり、
  シミュレータを起動し直しても直らなかった。原因は、前の `hinge set` が時間内に読み返せなかったときに残った
  quarantine の記録（理由は `uncertainHingeDelivery`）で、**エラーの文には quarantine と出ない。**
  `$TMPDIR/axe-hid-<uid>/*.quarantine.json` を開き、`simulatorUDID` がその Duo で `reason` が `uncertainHingeDelivery` なら、消すと戻る。
  `axe displays` は、このあいだも読めた

## mobilebuildmcp

**AXe の場所を指定しないと、操作系のツールは「成功」と返しても効かない。** 返り値を信用しない。
`tap` / `swipe` / `key_press` / `snapshot_ui` などは、何も指定しないと同梱の AXe
（`/usr/local/lib/node_modules/mobilebuildmcp/bundled/axe`、2026-11-21 時点で 1.8.0）を呼ぶ。
mobilebuildmcp は AXe を「`axePath` の設定 → 同梱版 → PATH」の順で探すので、PATH に Homebrew 版があっても同梱版が使われる。

**`MOBILEBUILDMCP_AXE_PATH=/opt/homebrew/bin/axe` を指定すると届く。** AXe は Homebrew 版を入れておく。

```bash
brew trust --formula cameroncooke/axe/axe && brew install axe
# → /opt/homebrew/bin/axe
```

MCP として使うときは、`~/.claude.json` の mobilebuildmcp の `env` に足す。

```json
"env": {
  "MOBILEBUILDMCP_AXE_PATH": "/opt/homebrew/bin/axe"
}
```

**足したあとは Claude アプリを起動し直す**（新しいセッションでもよい）。動いている MCP は起動したときの設定のままで、
効いているかは `ps eww -p <pid>` に `MOBILEBUILDMCP_AXE_PATH` が出るかで分かる。

- コマンドとして呼ぶときは `MOBILEBUILDMCP_AXE_PATH=/opt/homebrew/bin/axe mobilebuildmcp ui-automation <命令> --simulator-id <UDID> ...`
- **`tap` は座標ではなく `elementRef` を指定する。** `elementRef` はその時点の `snapshot_ui` の出力のものを使い、画面が変わったら取り直す
- 検索欄のような入力欄は `typeText` の対象としてだけ出て、`tap` の対象には入らないことがある。そのときは `axe tap` で押す
- `snapshot_ui` は前面の画面を読む。アプリの切り替え画面が開いたままだと、それを返す。ホームに戻してから取り直す
- コマンドの `tap` / `batch` / `swipe` のあとに「画面の読み直しが時間内に落ち着かなかった」という警告が出ることがある。操作は届いている

実測（mobilebuildmcp 2.7.1。MCP は iPhone 18 Pro、コマンドは iPad Pro 11-inch (M5) と iPhone 18 Pro）:

| 試したもの | 結果 |
|---|---|
| MCP の `snapshot_ui` | 要素の一覧と `elementRef` が返った |
| MCP の `tap`（「設定」の「一般」） | 届いた。見出しが「一般」に変わった |
| MCP の `key_press`（42） | 届いた。検索欄の値が1文字減った |
| コマンドの `batch`（スイッチ2つ） | 届いた。1回で2つとも切り替わった |
| コマンドの `swipe` | 届いた。一覧が 300pt 以上動いた |
| コマンドの `long-press`（ホーム画面のアイコン、1500ms） | 届いた。「ホーム画面を編集」のメニューが出た |
| コマンドの `type-text` | 届いた。ただし日本語入力のローマ字変換で `wifi` が `ウィふぃ` になった |

## 座標の落とし穴

- 横向きのときは、`describe-ui` の `frame` は横向きの座標（iPhone 18 Pro なら 874×402）で返り、
  パネルの座標の範囲と一致する
- **iPhone の「設定」アプリは横向きに回らない。** 横向きの試験は Safari など回るアプリで行う

## 背景 — Device Hub と devicectl

Xcode 27 で `Simulator.app` が無くなり、`Device Hub`（`<Xcode>.app/Contents/Applications/DeviceHub.app`、
bundle id は `com.apple.dt.Devices`）に置き換わった。端末ごとの別ウインドウではなく、左に端末の一覧、
右にその詳細という1ウインドウで、**シミュレータと実機を同じ画面で扱う。**

それに合わせて、実機用の CLI だった **`devicectl` がシミュレータも扱う**ようになった。
`simctl` に無い回転と可変ウインドウの幅が `devicectl` にあるのはこのため。
新しく何かをしたいときは、`simctl` だけでなく `devicectl` も見る。

一方で、ツールによっては対応が追いついておらず、成功と返すのに効かないものがある（上の「mobilebuildmcp」）。

参考: [Michael Tsai — Xcode 27's Device Hub](https://mjtsai.com/blog/2026/06/25/xcode-27s-device-hub/) /
[Bitrise — WWDC 2026: Device Hub and what it means for CI/CD](https://bitrise.io/blog/post/wwdc-2026-device-hub-and-what-it-means-for-ci-cd)
