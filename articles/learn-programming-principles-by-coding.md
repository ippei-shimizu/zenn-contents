---
title: "コードを書きながら学ぶ プログラミングの原理原則"
emoji: "🧭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['KISS', 'YAGNI', 'DRY', 'SLAP', 'OCP']
published: true
---

## はじめに

事業会社でエンジニアをしている[いっぺい](https://x.com/ippei_111)です。最近、転職をして以前よりも規模の大きなサービスの開発に携わることになりました。まだ転職して数ヶ月しか経ってはいませんが、1つのコードに対して求められる品質が上がり、自分の考えや知識がまだまだ足りていないと感じることが多いです。

プルリクエストのレビューを通して、要件を満たして動くコードは書けているが、「もっとこう書いた方がよいのでは？」という視点のコメントをもらうことも多く、将来的な変更に耐えられるか・保守性は良いか・読みやすいコードになっているかといった観点で、まだまだ力不足だと感じています。

そこで、改めて「プログラミングの原理原則」について体系的に学び直したいと考え、その学習のアウトプットとして本記事を書いています。

また、AIが大量のコードを書く時代だからこそ、そのアウトプットの良し悪しを判断する物差しとして「プログラミングの原理原則」のような、どうコードを書くべきか・設計すべきかは引き続き大切だと感じています。

## コードは必ず変更されることを前提に書く

### コードは読みやすさを重視して書く

当たり前ではありますが、書いたコードは将来的にほぼ確実に変更されます。
バグの修正・機能追加・ビジネス要件の変化などの様々な理由で、コードの変更が必要になり、その都度複数人の開発者の手によって書き換えられていきます。

つまり、コードを書く自分と、コードを読む人は別人になることが多くなります。そのため、コードは「書く時間」よりも「読む時間」の方が圧倒的に長く、コードを書く時は、書きやすさも重要ですが、読みやすさに重きを置いて書くことが大切になります。

### コードの良し悪しを判断するのは、現時点ではまだ人間の仕事である

コードの読みやすさを重視して書くという前提は、今のAIがコードを書いてくれる時代でも重要だと思っています。

AIによってコードを書く速さは格段に早くなりましたが、その出力されたコードが「変更に耐えられるのか」「将来の自分や他の開発者が読んで理解できるか」などの判断をするのは、2026年5月現在では、まだ人間が行う必要があります。
そのため、AIがコードを書く時代においても、「これは良いコードなのか」を判断する力は必要だと思っています。

本記事では、下記の原理原則について解説していきます。

![](/images/programming_principles_overview.png)

## KISS / YAGNI（コードをシンプルに保つ）

コードを書くときには、「シンプルさを保つ」ことが大切です。シンプルなコードは、コードを読む側にとっても読みやすく、変更にも強くなります。

しかし、「将来のために汎用的にしておこう」「もしかしたら必要になるかもしれない」などといった先回りや新しく学んだ記述や書き方を採用した結果、コードが複雑化してしまうケースがあります。

このシンプルさを保つための原則として「KISS（Keep It Simple, Stupid）」と「YAGNI（You Aren't Gonna Need It）」があります。

### KISS
KISS（Keep It Simple, Stupid）は、コードを書く時にできるだけシンプルさを保ち、ひとつの要素に持たせる責務を必要最小限に絞る、という考え方です。

シンプルなコードには、自分が日々のコードレビューでも実感する以下のような利点があります。
- 何をしているかが一目で把握できるので、読み手の理解が早い。
- 修正の影響範囲が小さく、変更しやすい。
- 入出力が単純で、テストが書きやすい。

よくありがちなのは、新しく覚えた技術や書き方を使ってみたいという気持ちで、必要以上におしゃれなコードを書いてしまうことです。（AIにコードを生成させた時もたまにある気がします）
本当にその複雑さが必要なのかは冷静に考える癖をつけるべきです。

![](/images/kiss_concept.png)

#### コードで見る
注文の合計金額（税込）を計算する `Order` クラスを例にします。「責務を細かく分けた方が綺麗だろう」とリファクタリングを進めた結果、こうなったとします。

##### Before: メソッドを過剰に分割してしまったコード
```rb
class Order
  TAX_RATE = 0.10

  def total_with_tax
    apply_tax(calculate_subtotal)
  end

  private

  def calculate_subtotal
    sum_item_prices
  end

  def sum_item_prices
    items.sum { |item| price_of(item) }
  end

  def price_of(item)
    item.price * quantity_of(item)
  end

  def quantity_of(item)
    item.quantity
  end

  def apply_tax(amount)
    amount * tax_multiplier
  end

  def tax_multiplier
    1 + TAX_RATE
  end
end
```

**何が問題か**
- 読み手が追跡すべきメソッド数が多すぎる
  - `total_with_tax` を理解するのに、`apply_tax` → `calculate_subtotal` → `sum_item_prices` → `price_of` → `quantity_of` … と、順番に7つのメソッドを行ったり来たりして読む必要がある。
- 意味のない委譲が混ざっている
  - `quantity_of(item)` は `item.quantity` を返すだけ、`calculate_subtotal` は `sum_item_prices` を呼ぶだけで、メソッドが層を1つ追加する以上の役割を果たしていない。
- 修正時に影響範囲が分かりにくい
  - 例えば「単価に値引きを反映したい」となった時、 `price_of` を直すべきか、`sum_item_prices` を直すべきか、判断に余計な負荷がかかってしまう。

このように、「メソッドを細かく分けた方がきれい」という気持ちが、結果として `Order` クラスを追いにくくしてしまっています。

##### After: 意味のある単位だけにメソッドを残す
```rb
class Order
  TAX_RATE = 0.10

  def total_with_tax
    subtotal * (1 + TAX_RATE)
  end

  private

  def subtotal
    items.sum { |item| item.price * item.quantity }
  end
end
```
`total_with_tax` の本体を読むだけで、「小計に税率を掛けている」と全体像が掴めるようになりました。 `subtotal` は「 `items` を巡って合計する」というロジックがあるので切り出す価値がありますが、それ以上の分割は追跡コストを増やすだけです。

#### まとめ
- 「このメソッドを切り出すことで、本当に読みやすくなっているか？」と意識してコードを書く必要がある。
- `item.quantity` を返すだけのような薄いラッパーや、一度しか呼ばれないメソッドは、インライン化した方が読みやすいことが多い。
- 「分けることが綺麗」ではなく、「読み手の認知負荷が下がっているか」を判断軸にしてコードを書く。

### YAGNI
YAGNI（You Aren't Gonna Need It）は、未来の需要を予測して先回りで実装するのではなく、必要だと確信できた時にそのコードを書く、という考え方です。

YAGNIはKISSと兄弟原則ですが、「先回りの拡張ポイント」に焦点を当てた原則です。
コードを書きながら、「将来的にこの機能を追加することになりそうだから、今のうちに引数を増やしておこう」「他の権限も増えるはずだから、メソッドを揃えて作っておこう」といった先回りで実装してしまうことは、YAGNIに違反してしまいます。

大体このような先回りをした予測は外れます。そして、予測のために書かれたコードは以下のようなデメリットを発生させます。
- 使われないまま保守し続ける負債
  - 動作確認・テスト・リファクタリングのたびに、使われていないコードが足を引っ張ってしまう。
- 読み手の認知負荷
  - 「このメソッド、いつ使われるのだろうか？」と考える時間が発生してしまう。
- 本来の責務のぼやけ
  - クラスが何のために存在するのかが、追加された未使用メソッドによって埋もれてしまう。

![](/images/yagni_concept.png)

#### コードで見る
ユーザーの権限判定を行う `User` モデルを例にします。先回りして、ロールごとに判定メソッドを揃えてしまった例です。

##### Before: 「将来のため」のロール判定メソッドが大量に並ぶ
```rb
class User < ApplicationRecord
  ROLES = %w[viewer editor moderator admin super_admin].freeze

  def viewer?
    role == "viewer"
  end

  def editor?
    role == "editor"
  end

  def moderator?
    role == "moderator"
  end

  def admin?
    role == "admin"
  end

  def super_admin?
    role == "super_admin"
  end

  def can_edit?
    editor? || admin? || super_admin?
  end

  def can_moderate?
    moderator? || admin? || super_admin?
  end

  def can_manage_users?
    admin? || super_admin?
  end
end
```

**呼び出し側**
```rb
# プロダクションで実際に呼ばれているのは admin? だけ
if current_user.admin?
  redirect_to admin_dashboard_path
end
```

**何が問題か**
- データとコードが食い違っている
  - `ROLES` には5種類のロールが並んでいるが、`users.role` カラムに実在するのは `member` と `admin` だけ。読み手は「 `viewer` や `moderator` はどこからくるのか」を探すコストが発生してしまう。
- 未使用メソッドの判断コストが永続化する
  - `viewer?` `editor?` `moderator?` `super_admin?` は呼ばれておらず、コードがあるだけで、リファクタリングのたびに「これ消していいのか？」を毎回考えることになってしまう。
- 組み合わせメソッドが予測ベースで書かれている
  - `can_edit?` `can_moderate?` `can_manage_users?` は「将来こういう組み合わせで権限を判定したい」という想像で書かれている。そのため、実際に必要になった時に、その組み合わせのまま使用される保証はない。
- 本来の責務がぼやける
  - `User` モデルを見た人は「権限管理が複雑なクラスだ」と感じてしまう。

##### After: 今、本当に呼ばれているものだけを残す
```rb
class User < ApplicationRecord
  def admin?
    role == "admin"
  end
end
```

こうすることで、「今のサービスは管理者かどうかしか区別していない」という事実が、コードからはっきりとわかるようになりました。

将来本当に権限が増えた時には、その時点で適切な形を選び直せばよいです。例えば、 `editor` ロールを追加するとなった時、本当にやるべきことは `editor?` メソッドを1つ追加することかもしれないし、Punditのような権限管理gemを導入することかもしれないし、`role` カラムを再設計することかもしれません。
要件が見えていない段階で先回りして書いたコードは、要件が見えた時にはたいてい捨てて書き直すことになるので、先回りして予測でコードを書くことはやめた方が良いです。

#### まとめ
- このメソッド・引数・分岐は、呼び出されているのか？を確認する。呼ばれていないなら、将来必要になったタイミングで書けば良いと判断できるようにする
- 本当にその実装が必要になった時に、設計を考える。要件が見えていない段階で先回りでコードを書かない。

## DRY / 参照の一点性（コードの重複を排除する）
KISSとYAGNIが「シンプルに保つ」ための原則だったのに対して、ここで扱う2つの原則は「重複を作らない」ための原則です。
普段コードを書いている中で「この条件どこかで書いたな」「同じカラムの判定が別のメソッドにもあるな」などと感じる場面があると思います。

DRY（Don't Repeat Yourself）は「同じロジックやコードを複数箇所に書かない」という観点で、参照の一点性は「1つの事実を1箇所だけで定義する」という観点です。

### DRY
DRY（Don't Repeat Yourself）は、同じロジックや条件・パターンを複数箇所に書かない、ということです。重複したコードがあれば、関数化・モジュール化・定数化などの手段で1箇所に集約します。

重複したコードを残しておくと、以下のようなコストが発生します。
- 読むコストが増える
  - 同じ意味のコードを何度も読み解く必要があり、本当に同じ意図なのかを毎回確認する手間がかかる。
- 修正コストが増える
  - 仕様変更のたびに複数箇所で正確に変える必要があり、1箇所でも漏れるとバグの温床になる。
- テストコストが増える
  - 同じロジックを複数箇所でテストする必要があり、テストが冗長になる。

ただし、DRY を機械的に適用するのは危険です。見た目が似ているだけで変更理由が異なるロジックを無理にまとめてしまうと、後で別々に変更したい時に苦しむことになります。「同じロジックに見えても、変更の理由が同じか？」を確認してから集約するのが安全です。

![](/images/dry_concept.png)

#### コードで見る
ユーザーの権限判定を行う `User`モデルを例にします。「投稿の編集」「投稿の削除」「コメント」と複数のメソッドがありますが、いずれも「アクティブで、BANされていない」という前提条件を共通で持っているケースです。

##### Before: 同じ前提条件が複数メソッドに散らばる
```rb
class User < ApplicationRecord
  def can_edit_post?(post)
    return false unless active?
    return false if banned_at.present?
    post.author == self || admin?
  end

  def can_delete_post?(post)
    return false unless active?
    return false if banned_at.present?
    post.author == self || admin?
  end

  def can_comment?(post)
    return false unless active?
    return false if banned_at.present?
    !post.locked?
  end
end
```

**何が問題か**
- 同じ前提条件が3メソッドに重複している
  - `active?` と `banned_at.present?` の2つのチェックが、3つのメソッドそれぞれに書かれている。読み手は「全メソッドで本当に同じチェックをしているのか？」を確認するために、3箇所を見比べる必要がある。
- 仕様変更時に修正漏れが起きやすい
  - 例えば、「一時停止状態のユーザーも操作できないようにする」という要件が来た時、3箇所すべてに修正を入れる必要がある。1箇所でも漏れると、特定の操作だけ抜け道ができてしまうバグになる。
- 各メソッドの本来の責務にノイズが入る
  - `can_edit_post?` の本来の責務は「投稿者本人か管理者かを判定すること」であり、共通の条件が入ってしまうことにより、ノイズになっている。

##### After: 共通の前提をprivateメソッドにまとめる
```rb
class User < ApplicationRecord
  def can_edit_post?(post)
    return false unless eligible?
    post.author == self || admin?
  end

  def can_delete_post?(post)
    return false unless eligible?
    post.author == self || admin?
  end

  def can_comment?(post)
    return false unless eligible?
    !post.locked?
  end

  private

  def eligible?
    active? && banned_at.blank?
  end
end
```
共通の前提条件を `eligible?` という1つのメソッドに集約しました。「アクティブで、BANされていない」という条件に名前が付き、各メソッドの本来の責務が全面にでるようになりました。これで、仕様変更があった時も、修正するのは `eligible?` の1箇所だけで済みます。

#### まとめ
- 同じロジックや条件が複数箇所に登場したら、抽象化できないかを検討する。
- 抽象化の最大の価値は「1箇所で済む」ことよりも、「修正漏れによるバグを防げる」こと。
- 共通条件を切り出すと、各メソッドの本来の責務が前面に出て、コードの意図が読みやすくなる。

### 参照の一点性
参照の一点性（Single Point of Reference）は、ある事実は1箇所だけに定義し、以後はそれを参照する、という考え方です。
DRYと近い概念ですが、DRYが「同じロジック・コード」の重複に焦点を当てるのに対して、参照の一点性は「データや状態をどこで管理するか」という、データ側の観点に焦点を当てた原則です。

同じ事実を複数箇所で持つと、以下のような問題が発生します。
- 整合性が崩れた状態が発生しうる
  - 片方はアクティブ、もう片方は削除済みといっているような、矛盾した状態がデータ上で生まれる。
- 読み手が「どっちが正？」と考える必要がある
  - 同期が取れているのか、片方だけ信じればいいのか、状態を読むたびに余計な判断が必要になる。
- 更新時に漏れが起きやすい
  - 状態を変えるたびに複数箇所を同時に更新する必要があり、片方を忘れると不整合になる。

![](/images/single_point_of_reference_concept.png)

#### コードで見る
ユーザーの「削除されたか」を表現する `User` モデルを例にします。`active` という booleanカラムと、`deleted_at` という timestampカラムの両方で「削除されたか」を表現してしまっているケースです。

##### Before: 同じ事実が2つのカラムに分散している
```rb
class User < ApplicationRecord
  # active: boolean
  # deleted_at: datetime

  def active?
    active && deleted_at.nil?
  end

  def deleted?
    !active || deleted_at.present?
  end

  def soft_delete!
    update!(active: false, deleted_at: Time.current)
  end

  def restore!
    update!(active: true, deleted_at: nil)
  end
end
```

**何が問題か**
- 真実の出所が2箇所ある
  - `active` カラムと `deleted_at` カラムの両方が「削除されたか」を表現している。どちらが真実なのか定まっていない。
- 矛盾した状態がデータ上で起こりうる
  - `active = true` かつ `deleted_at = "2024-01-01"` のような、論理的にありえない状態がデータベース上に発生してしまう可能性があり、バグやデータ移行で事故りやすい。
- 更新が2倍の手間になる
  - `soft_delete!` でも `restore!` でも、2つのカラムを必ず両方更新しないといけない。片方の更新漏れがあると、不整合状態になる。
- 派生メソッドの判定式が複雑になる
  - `active?` も `deleted?` も「2つのカラムを見て総合判断する」ロジックになっている。本来「削除されたか」というシンプルな問いに対して、複雑な判定が必要になっている。

##### After: deleted_atだけを真実とし、activeは派生で表現
```rb
class User < ApplicationRecord
  # deleted_at: datetime のみ
  # active カラムは廃止

  def active?
    deleted_at.nil?
  end

  def deleted?
    deleted_at.present?
  end

  def soft_delete!
    update!(deleted_at: Time.current)
  end

  def restore!
    update!(deleted_at: nil)
  end
end
```
`deleted_at` だけを「削除されたか」の真実として置きました。`active?`も`deleted?`も、そこから派生して計算するメソッドになっています。データベース上で矛盾した状態が発生する可能性がなくなり、`soft_delete!`も`restore!`も1つのカラムを更新するだけで済むようになりました。

#### まとめ
- 同じ事実を表現するカラム・属性・変数は、1箇所だけにする。
- 派生情報は「真実の出所」から計算して返すメソッドで表現すると、整合性は自動的に保たれる。
- 真実の出所が複数あると、矛盾した状態がデータ上で発生し、更新漏れによる不整合バグの温床になる。

## SLAP（抽象化レベル揃える）
KISSとYAGNIは「シンプルに保つ」、DRYと参照の一点性で「重複を作らない」を扱ってきました。次はもう少し別の角度から「読みやすさ」を支える原則を見ていきます。

KISSの章では「メソッドを過剰に分割すると追跡コストが上がる」という話をしました。一方で、メソッドを全く分けずに1つの長いメソッドに処理を詰め込むと、今度は「いまどの段階の話をしているのか」が読み取れなくなります。意味のある単位で分けるための原則が「SLAP」です。

SLAP（Single Level of Abstraction Principle）は、1つの関数の中では抽象化レベルを揃える、ということです。具体的には、高レベルの「何をしているか（What）」と、低レベルの「どうやっているか（How）」を、同じ関数の中に混ぜないように書きます。

抽象化レベルを揃えると、以下のメリットがあります。
- 上位関数を読むだけで処理の全体像が掴める
  - 関数の本体がそのまま「目次」のようになる。
- 認知負荷が下がる
  - 詳細を読みたいときだけ下位の関数に降りていけば良いので、読み手が抽象化レベルを切り替える回数が減る。
- 修正範囲を見極めやすくなる
  - 「どの抽象化レベルで何が起きているか」が明確になるので、変更すべき場所を素早く特定できる。

抽象化レベルが揃った関数は、他の関数を呼び出すコードだけで構成された複合関数になります。複合関数は処理を直接行わず、単に「次にこれをして、次にこれをする」と並べているだけのように見えますが、それこそが「目次」としての価値を持ちます。

![](/images/slap_concept.png)

### コードで見る
ECサイトの注文確定処理を行う `Order#confirm!` メソッドを例にします。在庫の確認・減算、ポイント付与、ステータス更新、通知...と、複数の処理を1つのメソッドに書いた結果、こうなったとします。

#### Before: 高レベルと低レベルが1つのメソッドに混在
```rb
class Order < ApplicationRecord
  def confirm!
    # 在庫チェック
    items.each do |item|
      stock = Stock.find_by(product_id: item.product_id)
      raise "Out of stock" if stock.nil? || stock.quantity < item.quantity
    end

    # 在庫減算
    items.each do |item|
      stock = Stock.find_by(product_id: item.product_id)
      stock.update!(quantity: stock.quantity - item.quantity)
    end

    # ポイント付与
    points = (subtotal * 0.01).to_i
    user.update!(points: user.points + points)

    # ステータス更新
    update!(status: "confirmed", confirmed_at: Time.current)

    # Slack通知
    SlackClient.new.post(text: "注文 ##{id} が確定")
  end
end
```

**何が問題か**
- 抽象化レベルが大きく異なる処理が同じ関数に並んでいる
  - 「在庫を確認する」「ポイントを付与する」という高レベルの意図と、「`Stock.find_by(product_id: ...)`」「`(subtotal * 0.01).to_i`」のような低レベルの実装詳細が、同じインデントで並んでしまっている。読み手は「いま何をしているのか」を理解するために、毎行抽象化レベルを切り替えながら読む必要がある。
- 全体像が把握しづらい
  - `confirm!` という名前の処理が、結局「どんなステップで」確定されるかは、コメントを目印にしないと読み取れない。コメントが消えたら、最初から読み解くしかない。
- コードがコメントに頼った構造になっている
  - 「在庫チェック」「在庫減算」「ポイント付与」などのコメントがないと、各ブロックの境界すらわかりにくい。コードそのものが意図を語れていない状態になっている。

#### After: confirm! が「目次」のように読める
```rb
class Order < ApplicationRecord
  def confirm!
    ensure_stock_available!
    decrement_stock
    award_points
    mark_as_confirmed
    notify_confirmation
  end

  private

  def ensure_stock_available!
    items.each do |item|
      stock = Stock.find_by(product_id: item.product_id)
      raise "Out of stock" if stock.nil? || stock.quantity < item.quantity
    end
  end

  def decrement_stock
    items.each do |item|
      stock = Stock.find_by(product_id: item.product_id)
      stock.update!(quantity: stock.quantity - item.quantity)
    end
  end

  def award_points
    points = (subtotal * 0.01).to_i
    user.update!(points: user.points + points)
  end

  def mark_as_confirmed
    update!(status: "confirmed", confirmed_at: Time.current)
  end

  def notify_confirmation
    SlackClient.new.post(text: "注文 ##{id} が確定")
  end
end
```

![](/images/slap_abstraction_levels.png)

`confirm!` メソッドの中身を読むだけで、「在庫を確認 → 減算 → ポイント付与 → ステータス更新 → 通知」という確定処理の全体像が掴めるようになりました。メソッド名そのものが処理の意図を表す形になっています。

各メソッドの詳細を読みたい場合は、対応する `private` メソッドに降りていくだけです。

ここで、KISSの章で扱った「過剰なメソッド分割」との違いを補足しておきます。KISSでは、`quantity_of(item)` のように `item.quantity` を返すだけの意味のないラッパーを作ることを避ける、という内容でした。一方、SLAPは、意味のある単位（処理の1ステップに対応する単位）でメソッドを切り出すことを求めます。

なお、本記事ではSLAPの説明に集中するため、`Stock.find_by` の重複や、複数回の `update!` のトランザクション処理については踏み込んでいません。実際の実装では、`Stock` の取得を1回にまとめる、`confirm!` 全体を `ActiveRecord::Base.transaction` で囲んで途中で失敗した時にロールバックする、といった改善が別途必要になります。

### まとめ
- 1つの関数の中では、抽象化レベルを揃える。「何をしているか」と「どうやっているか」を同じ関数に混ぜない。
- 抽象化レベルが揃った関数は「目次」のように読め、コメントなしで全体像が掴めるようになる。

## 名前重要（名前でコードの意図がわかるように）
ここまで、KISS/YAGNI/DRY/参照の一点性/SLAPを扱ってきましたが、いずれも「コードをどう構造化するか」という観点での原則でした。一方、「名前重要」はコードの中で使う名前そのものをどうつけるかに焦点を当てた原則です。

コードを書く時に、自分が一番悩むのが「このメソッドは何という名前にすべきか」「この変数名で意図が伝わるか」という命名です。それくらい、命名はコードの読みやすさを大きく左右します。関数名・変数名・クラス名を読むだけで「何をしているか」「何を返すか」「どう使うか」が伝わるように書ければ、その名前自体が短い説明文として機能します。

良い名前が付くと、関数の中身を毎回開かなくても、呼び出し側のコードを読むだけで処理の流れが理解できるようになります。逆に名前が曖昧だと、読み手は中身に降りて実装を読み解かないと意図が掴めません。

実際、自分自身もプルリクエストのレビューで「このメソッド名だと何をしているのか分かりにくい」と指摘されることが多く、名前の付け方が設計の良し悪しに直結すると感じる場面が増えています。命名に時間をかけて考えることは、そのコードが何のために存在するのか・どう使うべきかを、自分の中で整理することにも繋がります。

### コードで見る
ユーザーを表す `User` モデルを例にします。実装としては動作するものの、4つのメソッドそれぞれに命名上の引っかかりがある状態です。

#### Before: 複数の命名問題が混在
```rb
class User < ApplicationRecord
  def self.filter_by_role_and_active
    where(role: "admin", active: true)
  end

  def not_banned?
    banned_at.nil?
  end

  def process(amount)
    self.balance -= amount
    save!
  end

  def calc(p)
    p.price * 0.9
  end
end
```

呼び出し側のコードはこのようになります。
```rb
admins = User.filter_by_role_and_active
if user.not_banned?
  user.process(1000)
end
discounted = user.calc(product)
```

**何が問題か**
- 効果ではなく、実装手段で命名している（`filter_by_role_and_active`）
  - 「どう実装したか（roleとactiveでフィルタする）」を表しているだけで、「結局このメソッドは何を返すのか」が伝わってこない。呼び出し側の `User.filter_by_role_and_active` を読んでも、返ってくるのが何のリストかが曖昧。「どう実装したか」ではなく「何を返すか・何を達成するか」で命名したい。
- 否定形が読みづらさを生む（`not_banned?`）
  - 名前自体に否定を含むため、反対の意味を表すときは `!user.not_banned?` となり、二重否定で読み手の脳を疲れさせてしまう。最初から肯定形で書くべき。
- 動詞が曖昧で何をしているか伝わらない（`process`）
  - 「処理する」としか言っておらず、呼び出し側の `user.process(1000)` を読んでも、引き落としなのか入金なのか送金なのかがわからない。`process` `handle` `execute` のような曖昧な動詞は、自分もよくやってしまうパターン。
- 略称・1文字変数で意図が消える（`calc(p)`）
  - 何を計算しているのか、`p` が何を指しているのかが伝わらない。`p` は `product` か `price` か `payment` か、読み手の推測コストが上がる。

#### After: 名前で意図が伝わる
```rb
class User < ApplicationRecord
  def self.admins
    where(role: "admin", active: true)
  end

  def banned?
    banned_at.present?
  end

  def withdraw(amount)
    self.balance -= amount
    save!
  end

  def discounted_price(product)
    product.price * 0.9
  end
end
```
呼び出し側のコードは、名前を変えただけでこのように読みやすくなります。
```rb
admins = User.admins
unless user.banned?
  user.withdraw(1000)
end
discounted = user.discounted_price(product)
```
呼び出し側のコードを見るだけで、「管理者一覧を取得し」「BANされていなければ」「残高から引き落として」「商品の割引価格を計算する」という処理が、コメントなしで読み下せるようになりました。

### まとめ
- 名前は「短いコメント」のようなものであり、中身を開かなくても、呼び出し側だけで処理の流れが読み下せるように命名する。
- 「どう実装したか」ではなく、「何を返すか・何を達成するか」を表現する。否定形を避ける。曖昧な動詞を避ける。略称、1文字変数を避ける。

## OCP（拡張に開き、修正に閉じる）
OCPは「機能追加がきた時に、既存コードをどれだけ守れる設計になっているか」に焦点を当てた原則です。
OCP（Open-Closed Principle）は、コードを「拡張に対しては開いている、修正に対して閉じている」状態にしよう、という設計原則です。新しい機能を追加する時には、新しいコードを加えるだけで済み、すでに動いているコードには手を入れなくていい、という設計を目指すものです。

OCPが満たされていると、以下のような恩恵があります。
- 機能追加時に既存コードを変更しないため、リグレッションのリスクが下がる。
- レビュー範囲が新規追加部分に限定されるので、レビュアーの負荷も下がる。
- 「変更時の影響範囲」を読み手が脳内で追跡する負荷が減る。

逆にOCPが守られていないと、新しい振る舞いを追加するたびに既存コードに手を入れる必要が出てきます。最も典型的なのが、`if/elsif` の連鎖が増え続けるパターンです。新しい分岐を1つ足すために既存メソッドを開く、というのはOCP違反のサインの1つだと思います。

ただし、OCPはどこでも適用するべきものでないという点も同時に意識したいです。すべての分岐を最初からStrategyパターンで作り込むと、未使用の抽象化クラスが量産されて、YAGNIに違反します。

![](/images/ocp_concept.png)

### コードで見る
通知の送信処理を行う `NotificationSender` を例にします。Slack/Email/SMSの3つのチャンネルに対応していて、`channel` の値で送信先を分岐しているコードになります。

#### Before: if/elsif の連鎖で分岐
```rb
class NotificationSender
  def dispatch(notification)
    if notification.channel == "slack"
      SlackClient.new.post(notification.message)
    elsif notification.channel == "email"
      Mail.deliver(to: notification.email, subject: "通知", body: notification.message)
    elsif notification.channel == "sms"
      SmsClient.new.send(notification.phone, notification.message)
    end
  end
end
```

**何が問題か**
- 新しいチャンネルを追加するたびに `dispatch` メソッドを修正する必要がある
  - 例えば「アプリ内通知」を追加したくなったら、`elsif notification.channel == "in_app"` の分岐を `dispatch` メソッドに足すことになるので、動いているコードに手をいれる必要があり、リグレッションのリスクがともなう。
- 1つのメソッドに複数チャンネルの実装詳細が混ざっている
  - Slack のクライアント呼び出し、Mail の DSL、SMS のクライアント呼び出しが、同じメソッド内に並んでいる。
- テストの網羅が大変
  - チャネルが増えるたびに、`dispatch` メソッドのテストケースも増えていく。本来なら各チャネルごとに独立してテストできた方が望ましい。

#### After: チャネルごとに Strategy クラスへ
```rb
class NotificationSender
  CHANNELS = {
    "slack" => SlackChannel,
    "email" => EmailChannel,
    "sms"   => SmsChannel
  }.freeze

  def dispatch(notification)
    CHANNELS.fetch(notification.channel).new.deliver(notification)
  end
end

class SlackChannel
  def deliver(notification)
    SlackClient.new.post(notification.message)
  end
end

class EmailChannel
  def deliver(notification)
    Mail.deliver(to: notification.email, subject: "通知", body: notification.message)
  end
end

class SmsChannel
  def deliver(notification)
    SmsClient.new.send(notification.phone, notification.message)
  end
end
```

![](/images/ocp_extension_impact.png)

チャネルを `deliver(notification)` という共通インターフェースを持ったクラスに切り出しました。`NotificationSender#dispatch` は「`channel` の値に対応するクラスを選んで `deliver` を呼ぶだけ」というシンプルな構造になっています。

これで、新しいチャネル（例: in_app）を追加する時にやることは、以下の2つだけです。
1. `InAppChannel` クラスを新規作成して、`deliver(notification)` を実装する
2. `CHANNELS` ハッシュに `"in_app" => InAppChannel` を1行追加する

`NotificationSender#dispatch` 自体には一切手を入れません。拡張に対しては開いていて、修正に対しては閉じている状態が実現できました。

### まとめ
- 機能追加時に既存コードを変更しなくて済むように、共通インターフェース + Strategy で分岐を分離する。
- if/elsif の連鎖は OCP 違反のサイン。新しい分岐を足すたびに既存メソッドを開いているなら、Strategy 化を検討する。
- ただし OCPはどこにでも適用すべきではない。「変化が予想される部分」「実際に変化が起きた部分」に絞って遅延適用するのが、YAGNIとの両立を取るコツ。

## 単一責任（関心を分離する）
単一責任は、1つのクラスやモジュールに、複数の責務を持たせないという、設計の境界線をどこで引くかに焦点を当てた原則です。

単一責任の原則と関心の分離は、別々の原則として整理されることもありますが、「1つの単位に、複数の関心（変更理由）を混在させない」という点では共通しています。

一言で言うと、1つのクラス・モジュール・メソッドが担う責務は、1つだけにするということです。クラス名から「何をする責務のクラスか」が一言で答えられる状態を目指します。

責務を1つに絞ると、以下のようなメリットがあります。
- 変更時の影響範囲が小さくなる
  - そのクラスの変更理由が1つなので、修正が他の機能に波及しにくい。
- クラス名から責務が想像しやすくなる
  - 読み手が「結局このクラスは何をしているのか」を毎回探す必要がなくなる。
- テストが書きやすくなる
  - 1つの責務だけテストしたい時に、関係のない初期化やモックが不要になる。

**責務とは？**
「変更理由が同じかどうか」です。変更理由が同じものは1つの責務、違うものは別の責務と捉えると、責務の境界が見えやすくなります。

![](/images/single_responsibility_concept.png)

### コードで見る
ユーザーを表す `User` モデルが、パスワードリセットに関する一連の処理を全て抱えているケースを見ていきます。

#### Before: User モデルがパスワードリセットの全プロセスを抱える
```rb
class User < ApplicationRecord
  has_secure_password

  def reset_password!(new_password)
    self.password = new_password
    save!
    UserMailer.password_reset_confirmation(self).deliver_later
    AuditLog.create!(user: self, action: "password_reset", performed_at: Time.current)
    Rails.cache.delete("user_#{id}_session_token")
  end
end
```
**何が問題か**
- `User` モデルが複数の責務を抱えている
  - 「ユーザー情報の保存」「メール送信」「監査ログの記録」「セッションキャッシュの破棄」と、変更理由がバラバラの処理が `reset_password!` の中に並んでいる。`User` モデルが「ユーザーを表すモデル」というよりも「パスワードリセットの全プロセスを知っているクラス」に肥大化している。
- 各機能の変更が `User` モデルに波及する
  - メール文面を変える時、監査ログのフォーマットを変える時、キャッシュの仕組みを変える時、いずれも `User` モデルに手を入れることになる。
- テストが書きにくい
  - `reset_password!` をテストするとき、メール送信もキャッシュ破棄も監査ログ記録も発生してしまう。「パスワードを更新するだけ」のシンプルなテストが書けず、関係のないモックを用意する必要が出てくる。

#### After: ワークフローを Service に切り出し、User はコア責務に絞る
```rb
class User < ApplicationRecord
  has_secure_password
end

class PasswordResetService
  def initialize(user)
    @user = user
  end

  def call(new_password)
    @user.update!(password: new_password)
    notify_user
    record_audit
    invalidate_session
  end

  private

  def notify_user
    UserMailer.password_reset_confirmation(@user).deliver_later
  end

  def record_audit
    AuditLog.create!(user: @user, action: "password_reset", performed_at: Time.current)
  end

  def invalidate_session
    Rails.cache.delete("user_#{@user.id}_session_token")
  end
end
```

呼び出し側のコードはこのようになります。
```rb
PasswordResetService.new(user).call(new_password)
```

`User` モデルから `reset_password!` を取り除き、`has_secure_password` だけを残しました。パスワードリセットの一連のワークフロー（更新・通知・監査・キャッシュ破棄）は、`PasswordResetService` に集約されています。

これによって、
- メール文面を変える時は `notify_user` だけを修正する
- 監査ログの仕様を変える時は `record_audit` だけを修正する
- キャッシュの仕組みを変える時は `invalidate_session` だけを修正する

と、変更理由ごとに修正範囲が局所化されるようになりました。

### まとめ
- 1つのクラス・モジュール・メソッドが担う責務は1つだけにする。クラス名から「何をする責務か」が一言で答えられる状態を目指す。
- 責務の境界は「変更の理由が同じかどうか」で判断する。理由が違う処理は、本来は別のクラスに分けるべき関心。
- Fat Model になりがちな処理（ワークフロー的な複数ステップ）は、Service Object に切り出すと責務がきれいに分離される。

## 実装ではなくインタフェースに依存する
「実装ではなくインタフェースに依存する」とは、ビジネスロジック（高レベル）が、具体的な実装の詳細（低レベル）を直接知らない設計を目指すということです。
具体的なクラス名やAPIの呼び方ではなく、「こういうメソッドをもっている」という抽象に依存することで、実装の差し替えやテストが容易になります。

なお、JavaやTypeScriptのような言語では `interface` という言語機能がありますが、Rubyにはありません。Rubyでは「同じメソッド名を持つ別クラス」を差し替え可能にする、いわゆるダックタイピングで実質的なインタフェースを表現します。
本章で「インタフェース」と呼んでいるのも、このダックタイピング的な意味での「共通の振る舞いを持つクラス群」のことです。

実装ではなくインタフェースに依存することで、以下のメリットがあります。
- 実装の差し替えが容易になる
- テストが書きやすくなる
- ビジネスロジックが安定する

![](/images/interface_dependency_concept.png)

### コードで見る
画像をアップロードする `ImageUploader` を例にします。アップロード先として AWS S3 を使うコードが、S3のAPI呼び出しをクラス内に直接書いているケースです。

#### Before: AWS S3の実装に直接依存
```rb
class ImageUploader
  def upload(file, filename)
    s3 = Aws::S3::Client.new(region: "ap-northeast-1")
    s3.put_object(
      bucket: "my-app-images",
      key: filename,
      body: file
    )
    "https://my-app-images.s3.ap-northeast-1.amazonaws.com/#{filename}"
  end
end
```

**何が問題か**
- ビジネスロジックがS3の詳細を知っている
  - `ImageUploader` の本来の責務は「画像をアップロードしてURLを返す」というシンプルなもの。それなのに、S3のリージョン名・バケット名・SDKのメソッド呼び出しといった詳細を抱え込んでいる。
- 実装の差し替えができない
  - 開発環境ではローカルファイルに保存したい、テストではメモリ上に保存したい、といった要件が出てきても、S3の実装に密結合しているため切り替えられない。
- テストで本物のS3が必要になる
  - `ImageUploader` をテストするには、毎回 S3 のモックを用意するか、テスト用のバケットを用意する必要がある。これではビジネスロジックだけを純粋にテストできない。

![](/images/interface_dependency_direction.png)

#### After: Storageインタフェースに依存
```rb
class ImageUploader
  def initialize(storage:)
    @storage = storage
  end

  def upload(file, filename)
    @storage.save(filename, file)
  end
end

class S3Storage
  def initialize(bucket:, region:)
    @bucket = bucket
    @region = region
  end

  def save(filename, file)
    s3 = Aws::S3::Client.new(region: @region)
    s3.put_object(bucket: @bucket, key: filename, body: file)
    "https://#{@bucket}.s3.#{@region}.amazonaws.com/#{filename}"
  end
end

class LocalStorage
  def initialize(directory:)
    @directory = directory
  end

  def save(filename, file)
    path = File.join(@directory, filename)
    File.binwrite(path, file)
    "/local/#{filename}"
  end
end
```
`ImageUploader` は、`save(filename, file)` というメソッドを持つ何かしらのクラス（= Storage）に依存するようになりました。具体的に S3 を使うのか、ローカルファイルを使うのか、テスト用の Fake を使うのかは、`ImageUploader` の外側で決められます。

呼び出し側では、環境に応じて実装を差し替えます。
```rb
# プロダクション環境
ImageUploader.new(
  storage: S3Storage.new(bucket: "my-app-images", region: "ap-northeast-1")
).upload(file, "photo.png")

# 開発環境（ローカルファイル）
ImageUploader.new(
  storage: LocalStorage.new(directory: "tmp/uploads")
).upload(file, "photo.png")
```

テスト時には、ファイルへの書き込みすら発生しない Fake を差し込めます。
```rb
class FakeStorage
  attr_reader :saved_files

  def initialize
    @saved_files = {}
  end

  def save(filename, file)
    @saved_files[filename] = file
    "/fake/#{filename}"
  end
end

RSpec.describe ImageUploader do
  it "ファイルをアップロードしてURLを返す" do
    storage = FakeStorage.new
    uploader = ImageUploader.new(storage: storage)

    url = uploader.upload("dummy", "test.png")

    expect(url).to eq("/fake/test.png")
    expect(storage.saved_files).to have_key("test.png")
  end
end
```
`ImageUploader` の本来の責務である「ファイルとファイル名を受け取って、保存先の URL を返す」というロジックだけを、外部依存なしでテストできるようになりました。

### まとめ
- 高レベル（ビジネスロジック）が、低レベル（具体的な実装の詳細）を直接知らない設計を目指す。
- Rubyではダックタイピングを利用して、「同じメソッド名を持つ別クラス」を差し替え可能にすることで、インタフェース依存を実現する。

## 凝縮度・結合度

### 凝縮度
凝縮度は、1つのモジュールの中で、そこに含まれているすべての要素が同じ目的に貢献しているかを測る尺度です。
- 高凝縮: クラス内のメソッドや属性が、すべて1つの責務に向かって貢献している。
- 低凝縮: 関係のない処理が同じクラスに混在している。

![](/images/cohesion_concept.png)

低凝縮のクラスを見ていきます。
```rb
class User < ApplicationRecord
  has_secure_password

  def authenticate(password)
    BCrypt::Password.new(password_digest) == password
  end

  # 関係のない処理が混入している
  def total_revenue_in_yen
    orders.sum(&:total)
  end

  def generate_monthly_report_csv
    CSV.generate do |csv|
      csv << ["title", "amount"]
      orders.each { |o| csv << [o.title, o.total] }
    end
  end
end
```

`User` というクラス名なのに、認証・売上計算・CSVレポート生成という、互いに無関係な責務が混在しています。これは凝縮度が低い状態です。
前章で扱った単一責任の原則を守ると、自然と凝縮度の高いクラスになります。
```rb
class User < ApplicationRecord
  has_secure_password

  def authenticate(password)
    BCrypt::Password.new(password_digest) == password
  end
end

class UserRevenueCalculator
  def initialize(user)
    @user = user
  end

  def total_in_yen
    @user.orders.sum(&:total)
  end
end

class UserMonthlyReportCsv
  def initialize(user)
    @user = user
  end

  def generate
    CSV.generate do |csv|
      csv << ["title", "amount"]
      @user.orders.each { |o| csv << [o.title, o.total] }
    end
  end
end
```

それぞれのクラスが「1つの目的」だけを担うようになり、`User` クラスは認証関連だけに集中できるようになりました。

![](/images/cohesion_coupling_matrix.png)

### 結合度
結合度は、2つ以上のモジュールが、互いの内部にどれだけ深く踏み込んでいるかを測る尺度です。
- 密結合: 一方の変更が他方にも変更を強いる。互いの内部実装を知り合っている状態。
- 疎結合: インタフェースだけで会話していて、内部実装の変更が波及しない状態。

![](/images/coupling_concept.png)

疎結合に保つコツとしては、
- データの受け渡しは引数で行う（グローバル変数や共有状態を避ける）
- 相手のクラスの内部状態を直接書き換えない
- 振る舞いが、渡された値によって変わらない（ステートレスな設計）

といった指針があります。密結合の例を見ていきます。

```rb
class Notifier
  def notify(user, message)
    return if user.preferences[:notifications_enabled] == false
    return if user.banned_at.present?

    SlackClient.new.post(user.slack_account_id, message)
  end
end
```
`Notifier` が `User` の内部構造（`preferences` のキー、`banned_at` の存在）を直接知っているため、`User` 側でカラム名や条件を変えると `Notifier` も書き直す必要があります。これは結合度が高い状態です。

```rb
class Notifier
  def notify(user, message)
    return unless user.notifiable?
    user.send_notification(message)
  end
end

class User < ApplicationRecord
  def notifiable?
    preferences[:notifications_enabled] && banned_at.nil?
  end

  def send_notification(message)
    SlackClient.new.post(slack_account_id, message)
  end
end
```
`Notifier` は `User` の中身を知らずに、`notifiable?` と `send_notification` というインタフェースを通じて会話するようになりました。
`User` 側の内部実装が変わっても、`Notifier` は変更不要です。

### まとめ
- 凝縮度はモジュール内の純度、結合度はモジュール間の関係の密度を測る物差し。高凝縮・疎結合のコードを目指す。
- 凝縮度を高めるには単一責任、結合度を下げるにはインタフェースへの依存が、それぞれ具体的な手段になる。

## おわりに
今回、「[プリンシプル オブ プログラミング 3年目までに身につけたい 一生役立つ101の原理原則](https://www.shuwasystem.co.jp/book/9784798046143.html)」を読んで、KISS / YAGNI / DRY / 参照の一点性 / SLAP / 名前重要 / OCP / 単一責任 / 凝縮度・結合度 について、サンプルコードを使いながらまとめてみました。

個人的には、AIが大量にコードを生成してくれるようにはなりましたが、2026年5月現在の状況では、まだ人間が出力されたコードに対して「本当に良いコードなのか」を判断する必要があると思うので、今回学んだことを実際にコードを書く時にしっかり活かしていきたいです。

### 参考
- プリンシプル オブ プログラミング 3年目までに身につけたい一生役立つ101の原理原則

https://www.shuwasystem.co.jp/book/9784798046143.html
