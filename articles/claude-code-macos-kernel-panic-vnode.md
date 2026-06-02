---
title: "Claude Codeを起動するとMacがカーネルパニックする原因を調べたら、vnodeの枯渇だった"
emoji: "💥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['macOS', 'ClaudeCode', 'vnode', 'kernelpanic', 'Spotlight']
published: true
---

## はじめに

事業会社でエンジニアをしている[いっぺい](https://x.com/ippei_111)です。

先日、個人開発中に突然 Claude Code を起動するたびに Mac がカーネルパニックを起こして強制再起動する、という現象に遭遇しました。1日で4回も再起動を食らい、さすがに開発どころではなくなったので、原因をきちんと突き止めることにしました。

最初は「Claude Code が悪いのでは？」「macOSのアップデートのせいでは？」「ついにMacが壊れたか？」と色々疑ったのですが、一つひとつ切り分けていった結果、犯人は **vnode の枯渇** という、普段あまり意識しないOSのリソースでした。

同じように「特定のアプリを使うとMacごと落ちる」という現象に悩んでいる方の助けになればと思い、原因にたどり着くまでの調査の流れと対処をまとめてみます。

:::message
本記事はあくまで自分の環境で起きた事象と、その調査・対処の記録です。環境によって原因は異なる可能性があります。また、間違っている説明や解釈がありましたら、ご指摘いただけると幸いです。
:::

:::message
**カーネルパニックとは**
OS（macOS）の中核であるカーネルが、これ以上処理を続けると危険だと判断したときに、システム全体を強制的に停止させる仕組みです。Windowsでいう「ブルースクリーン」に近いもので、発生するとMacは強制的に再起動します。
:::

## 発生した環境

- MacBook Pro (Mac16,1) / Apple M4 / メモリ 24GB
- macOS Tahoe 26.5.1
- Claude Code 2.1.150

## 症状

ある日突然、Claude Code を起動するたびに **Mac が画面ごと落ちて勝手に再起動する** ようになりました。アプリがクラッシュするのではなく、OSごと強制的に再起動がかかります。前日まで普通に使えていたのに、急にです。

最初はターミナルが落ちているように見えたのですが、よく見ると毎回マシン全体が再起動していました。しかも Claude Code を起動した直後に高確率で発生します。

## まずはカーネルパニックのログを読む

OSごと落ちているので、まずはカーネルパニックのログを開いてみました。macOSでは、カーネルパニックが起きると `~/Library/Logs/DiagnosticReports/` に `.panic` という拡張子のログファイル（パニック時の状況を記録した診断レポート）が自動で保存されます。

開いてみると、毎回まったく同じ署名が記録されていました。

```
panic(cpu N caller ...): initproc exited -- exit reason namespace 2 subcode 0xa
Panicked task ...: pid 1: launchd
```

ここでキーになるのが `launchd`（ランチディー）と `initproc` です。

:::message
**launchd / initproc とは**
`launchd` は、macOSが起動して最初に立ち上がる、すべてのプロセスの親にあたる管理プロセスです。プロセスIDが `1`（PID 1）で、他のアプリやサービスはすべてこの `launchd` から生まれます。`initproc` はこの「最初のプロセス（＝launchd）」を指すカーネル内部での呼び名です。
:::

このログから読み取れたのは、

- `initproc exited` = **`launchd`（PID 1）が終了している**
- `launchd` はOSの根幹なので、これが死ぬとカーネルは「もう続行できない」と判断して **パニック → 強制再起動** する

ということでした。

つまり、ユーザーが使っているアプリが直接OSを落としているのではなく、**何らかの理由で `launchd` がリソースを確保できずに死に、その巻き添えでOSごと落ちている** という構図です。「何のリソースが足りなくなったのか」を突き止めるのが、この調査のゴールになりました。

:::message
panic文字列中の `namespace` / `subcode` の数値の意味は資料によって解釈が分かれるため、本記事では数値の断定は避け、「launchd が異常終了して `initproc exited` パニックに至った」という事実に絞って扱います。後述するとおり、その引き金は vnode の枯渇でした。
:::

![](/images/kernel-log.png)

## 容疑者を一つずつ潰していく

「Claude Code を起動したときだけ落ちる」とはいえ、いちアプリが直接OSを落とせるはずがありません。何が引き金になっているのかを、一つずつ切り分けていきました。

最初に疑ったものは、結論から言うとすべてシロでした。

| 疑ったもの | 検証方法 | 結果 |
| --- | --- | --- |
| セキュリティ系の拡張機能 | 無効化して再起動 | シロ（再発） |
| キーボード拡張（Karabiner） | 無効化して確認 | シロ |
| ターミナル固有の問題 | 別のターミナルでも起動 | シロ（別アプリでも落ちる） |
| 作業ディレクトリ内のデータ | 空ディレクトリで起動 | シロ（空でも落ちる） |
| ストレージ / ファイルシステムの破損 | `diskutil verifyVolume /`・SMART | シロ（問題なし） |
| ハードウェアの故障 | Apple Diagnostics | シロ（`ADP000`） |

:::message
上の表で使ったチェック方法の補足です。
- **`diskutil verifyVolume /`** … macOS標準のコマンドで、ディスク（ファイルシステム）に破損がないか検証します。
- **SMART** … ストレージ自身が持つ自己診断機能。ディスクの健康状態を確認できます。
- **Apple Diagnostics** … Mac内蔵のハードウェア診断ツール。起動時に特定のキーを押すと実行でき、メモリやSoCなどに異常がないかを調べられます。`ADP000` は「問題なし」を表す結果コードです。
:::

特に効いたのが、**「単純な高負荷」では落ちないことを確認できた** ことです。Claude Code は内部で git や ripgrep を多用するので、最初は「大量のプロセス生成やメモリ負荷で落ちているのでは」と考えました。そこで、負荷の種類を分けて単体で再現を試みました。

```bash
# CPUを全コア（10コア）全開で回す → 落ちない
for i in $(seq 1 10); do yes > /dev/null & done

# プロセスを大量にfork/exec → 落ちない
for i in $(seq 1 2000); do (/bin/echo hi > /dev/null) & done; wait

# 巨大ファイルを一気に読み込む（メモリへのpage-in） → 落ちない
dd if=/dev/zero of=/tmp/bigtest bs=1m count=8000
cat /tmp/bigtest > /dev/null
```

それぞれのコマンドが何をしているかを補足します。

- `for i in $(seq 1 10); do yes > /dev/null & done` … `yes` は同じ文字を延々と出力し続けるコマンドで、CPUを使い切ります。それを `&`（バックグラウンド実行）で10個同時に走らせ、全コアを全開にしています。
- `for i in $(seq 1 2000); do (/bin/echo hi > /dev/null) & done; wait` … プロセスを2000個まとめて起動し、短時間に大量のプロセス生成（fork/exec）が起きる状況を作っています。`wait` は起動した全プロセスの終了を待つ指定です。
- `dd if=/dev/zero of=/tmp/bigtest bs=1m count=8000` … `dd` はデータをコピーするコマンドで、ここでは8GBのファイルを作成しています。`cat ... > /dev/null` でそれを一気に読み込み、メモリへの大量読み込み（page-in）を発生させています。

:::message
**page-in とは**
ディスク上のデータをメモリに読み込む操作のことです。ここでは「メモリへ大量にデータを読み込む負荷」をかけて、それが原因で落ちないかを確認しています。
:::

CPUを全開で回しても、プロセスを2000個生成しても、8GBのファイルを一気に読み込んでも、まったく落ちませんでした。それなのに Claude Code を起動すると落ちる。

ここで「単純な負荷ではなく、もっと特定の種類のリソースを食い潰しているのでは」と当たりがつきました。

## 真因は vnode の枯渇だった

決め手になったのが、次のコマンドの結果でした。

```bash
sysctl kern.num_vnodes kern.maxvnodes
# kern.num_vnodes: 263168
# kern.maxvnodes: 263168
```

`sysctl` はカーネルの各種パラメータを確認・変更するコマンドです。ここでは現在使用中のvnode数（`num_vnodes`）と、その上限（`maxvnodes`）を表示しています。

**使用中の vnode 数が、上限にぴったり張り付いていました。** しかも、再起動してまだ5分しか経っていないのに、すでに上限に達していました。

### そもそも vnode とは

`vnode`（virtual node）は、**カーネルが「開いているファイルやディレクトリ」を管理するためのオブジェクト** です。APFSでもexFATでもネットワーク越しのファイルでも、種類の違うファイルシステムを共通の窓口で扱えるように抽象化する役割を持っています。ファイルにアクセスするたびに消費されます。

![](/images/vnode.png)

ここでのポイントは、vnode が **キャッシュ** だということです。ファイルを開いてvnodeを作るにはコストがかかるので、カーネルは **ファイルを閉じてもvnodeをすぐ捨てず、再利用に備えて保持** します。なので `num_vnodes` が上限近くまで埋まっていること自体は、実は異常ではなく正常な動作です。通常はカーネルが「上限に近づいたら古い未使用のvnodeを捨てて再利用」してくれるので、上限近くで使っていても問題なく回ります。

### なぜ枯渇するとカーネルパニックに至るのか

問題が起きるのは、**「上限まで使い切っていて、かつ"今まさに使用中"のvnodeが大半で、再利用に回せる空きが残っていない」** という状態になったときです。この状態で誰かが新しいvnodeを要求すると、確保に失敗します。

それを要求したのがPID 1の `launchd` だった場合、`launchd` が死に、`initproc exited` パニックでシステムごと落ちる、というわけです。これが今回のカーネルパニックの正体でした。

:::message
個人的には、ここはmacOS側の作りにも改善の余地があると感じています。本来「vnodeが作れない」という状況は、要求したプロセスにエラーを返すだけで済むはずで、それでOSごと落ちてしまうのは少し過剰な気がします。
:::

### これまでの切り分けと辻褄が合う

vnodeの枯渇だと考えると、ここまでの検証結果がすべて説明できました。

- **CPU全開・大量fork・大量page-inで落ちなかった** → これらはvnodeをほとんど消費しないから
- **Claude Codeでだけ落ちた** → git や ripgrep、ファイルウォッチなどでvnodeを要求する操作を高頻度で行うため、上限に張り付いた状態に「最後の一押し」を加えていた
- **アプリやディレクトリを問わず落ちた** → 引き金はvnodeの枯渇で、それらとは無関係だったから

つまり Claude Code は犯人ではなく、**すでに限界まで埋まっていたところに、たまたま最後の一押しを加えていただけ** だったのです。

![](/images/claude-code-only.png)

## では、誰がvnodeを食い尽くしていたのか

次に「何がvnodeをそんなに消費しているのか」を調べました。ここで役立ったのが `fs_usage` です。今まさに何がファイルシステムにアクセスしているかを、プロセス名つきでリアルタイムに表示してくれるコマンドです。

```bash
sudo fs_usage -w -f filesys 2>/dev/null | head -100
```

出力を眺めると、特定のプロセスが **`node_modules` の中の極小ファイルを1個ずつ延々と開き続けている** のが見えました。

```
.../node_modules/@react-native/.../third_party/lighthouse/...
.../node_modules/caniuse-lite/data/features/input-color.js
```

ファイルを開いていた主役は、**Spotlightのインデックス作成プロセス**（`mds_stores` / `mdworker_shared`）でした。

:::message
**Spotlight とは**
macOSに標準で備わっているファイル検索機能です（`⌘ + スペース` で出てくる検索窓）。高速に検索できるよう、裏側でファイルの中身を読み取って索引（インデックス）を作っており、その作成を担当する裏方のプロセスが `mds_stores` や `mdworker_shared` です。
:::

開発プロジェクトの `node_modules` にある数万〜数十万のファイルを片っ端からインデックスしようとして、vnodeを上限まで食い尽くしていたのです。

決定的だったのは、**Claude Code を終了させてもvnodeが増え続けた** ことです。これで「食っていたのは Claude Code ではなく Spotlight」だと確信できました。

### 「前日まで普通に使えていたのに突然」だった理由

vnodeの枯渇は、閾値を超えた瞬間に一気に表面化します。新しいプロジェクトを clone したり `npm install` で `node_modules` を大量に生成したりして、Spotlightが走査する対象のファイル数がある日ベースラインを押し上げ、「起動するだけで上限に張り付く」ラインに乗ってしまった、と考えると辻褄が合いました。

## 対処

「vnodeを食い潰す原因を断つ（根本対応）」と「vnodeを枯渇しにくくする（保険）」の2段構えで対処しました。

### 1. node_modules を Spotlight のインデックス対象から外す（根本対応）

そもそも `node_modules` をSpotlightの検索対象にするメリットはほとんどないので、開発リポジトリを置いている親フォルダごとインデックス対象から外します。

「システム設定 → Spotlight → 検索プライバシー」に、開発用のフォルダ（例: `~/projects`）を追加します。あるいは、各 `node_modules` に目印ファイルを置くことでも除外できます。

```bash
find ~/projects -type d -name node_modules -prune 2>/dev/null \
  | while read d; do touch "$d/.metadata_never_index"; done
```

`.metadata_never_index` という名前の空ファイルをフォルダに置いておくと、Spotlightはそのフォルダをインデックスしなくなります。

必要に応じて、`mdutil`（Spotlightのインデックスを管理するコマンド）で一度インデックスを停止・消去してから、除外設定を入れた状態で作り直します。

```bash
sudo mdutil -i off -a   # インデックス作成を停止
sudo mdutil -E -a       # 既存インデックスを消去
sudo mdutil -i on -a    # （除外を設定したうえで）必要なら再有効化
```

:::message
`num_vnodes` は再起動するまで下がりません（カーネルが上限まで確保したキャッシュ枠を自分から解放しないため）。走査しているプロセスを止めても数字は張り付いたままなので、最後に再起動してリセットするのが確実です。
:::

### 2. vnodeの上限を引き上げて恒久化する（保険）

枯渇しても余裕があるように、`kern.maxvnodes` の上限を引き上げておきます。まずは即時反映で試します。

```bash
sudo sysctl kern.maxvnodes=2000000
sysctl kern.num_vnodes kern.maxvnodes
```

ただし `sudo sysctl` で設定した値は再起動するとリセットされてしまうので、起動時に自動で適用されるよう LaunchDaemon を作成します。

:::message
**LaunchDaemon とは**
macOSが起動するタイミングで、指定したコマンドを自動的に実行してくれる仕組みです。`/Library/LaunchDaemons/` に設定ファイル（plist）を置いておくと、再起動のたびに毎回手で打たなくても設定が適用されます。
:::

```bash
sudo tee /Library/LaunchDaemons/limit.vnodes.plist > /dev/null <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>limit.vnodes</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/sbin/sysctl</string>
    <string>kern.maxvnodes=2000000</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
EOF

sudo chown root:wheel /Library/LaunchDaemons/limit.vnodes.plist
sudo launchctl load -w /Library/LaunchDaemons/limit.vnodes.plist
```

:::message
上限を上げると、その分だけカーネルが固定で使うメモリ（wired memory）が増える可能性があります（vnode 1個あたり、おおよそ1〜2KB程度）。メモリに余裕のあるマシン向けの対処です。また、極端に大きな値にするのは逆効果になり得るので、実際の使用量より少し上の値に留めるのが無難だと思います。そして何より、**上限を上げただけで根本（大量走査）を放置しない** ことが大切です。上限はあくまで保険で、走査を止めなければまた埋まります。
:::

### 対処後の確認

最後に、vnodeの推移を監視して、上限に向かって増え続けないことを確認します。

```bash
while true; do echo "$(date +%T) $(sysctl -n kern.num_vnodes)"; sleep 2; done
```

これは「2秒ごとに、現在時刻と使用中のvnode数を表示し続ける」ワンライナーです。`while true; do ... done` で無限ループを回し、`sleep 2` で2秒間隔をあけています。これを流しっぱなしにすることで、vnodeが増え続けるのか、横ばいで安定するのかをリアルタイムに観察できます（止めるときは `Ctrl + C`）。

対処後は **数 vnode/秒** 程度でほぼ横ばいになり、上限に対して十分な余裕を保ったまま安定しました（以前は短時間で上限に張り付いていました）。この状態であれば、新しいvnodeを要求しても枯渇せず、`launchd` が死ぬこともありません。

![](/images/vnode-graph.png)

これで Claude Code を起動しても落ちなくなり、無事に開発を再開できました。

## 調査に使ったコマンドまとめ

今回の調査で役立ったコマンドをまとめておきます。

```bash
# vnodeの現状と推移を確認
sysctl kern.num_vnodes kern.maxvnodes
while true; do echo "$(date +%T) $(sysctl -n kern.num_vnodes)"; sleep 2; done

# 今まさに何がファイルを大量に開いているかを実況
sudo fs_usage -w -f filesys 2>/dev/null | head -100

# プロセスごとに開いているファイル数を集計
sudo lsof 2>/dev/null | awk '{print $1}' | sort | uniq -c | sort -rn | head

# Spotlightのインデックス状態を確認
mdutil -s /

# 直近のパニックログを確認
ls -lt ~/Library/Logs/DiagnosticReports/ | head
ls -lt /Library/Logs/DiagnosticReports/ | head
```

`lsof` は「開いているファイル一覧（list open files）」を表示するコマンドです。上の例では、その出力をプロセス名で集計し、開いているファイル数が多い順に並べています（`awk` で1列目のプロセス名を取り出し、`sort | uniq -c` で件数を数え、`sort -rn` で多い順に並べ替え）。どのプロセスがたくさんファイルを開いているかの当たりをつけるのに使えます。

## おわりに

今回は「Claude Code を起動するとMacがカーネルパニックする」という現象を調べていったら、最終的に **Spotlightが `node_modules` を走査してvnodeを枯渇させ、その巻き添えで `launchd` が死んでいた** という、最初の見た目からは想像しにくい原因にたどり着きました。

調査を通して、改めて以下のことを学びました。

- OSごと落ちるときは、まず `.panic` ログの署名を読む。`initproc exited`（launchd の死）が出ていたら、リソースの枯渇を疑う。その代表格が vnode。
- 「アプリAを使うと落ちる」からといって、アプリAが犯人とは限らない。限界ギリギリの状態に最後の一押しをしているだけのこともある。負荷の種類を分けて単体で再現を試すと切り分けやすい。
- `vnode` はファイルアクセスに必須のリソースで、上限まで使うこと自体は正常。問題になるのは「再利用枠を食い尽くす大量走査」が起きたとき。
- `fs_usage` は「今まさに何がファイルを大量に触っているか」をプロセス名つきで見られて、こういうリソース枯渇系の原因特定にとても強い。
- `node_modules` はSpotlightの検索対象から外しておくと、インデックスの無駄もなくなり一石二鳥。

普段あまり意識しない低レイヤーのリソースが原因になることもあるのだと、良い勉強になりました。同じ現象に悩んでいる方の参考になれば嬉しいです。
