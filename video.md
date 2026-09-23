# Android 動画再生の実装・検証ガイド

この文書は、Galaxy Z Fold8 を含む Android 端末で、動画再生を実装するための共通指針である。Media3 による端末内再生を中心に、全画面、シーク、Google Cast、ピクチャー・イン・ピクチャー（PIP）、折りたたみ姿勢、外部表示、Bluetooth 設定と音声出力先切替への入口を扱う。コーディングエージェントは、アプリ固有の画面構成、メディアの保管先、再生順を既存の仕様から確認してから適用する。

初回は「再生状態を一つに保つ」から「入口や姿勢が変わっても操作の位置関係を保つ」までを読む。Bluetooth 設定と音声出力先切替の章も必ず確認し、最後に「実機で確認する」を使う。コンパクトな埋め込み表示、時間指定のスキップ、前後のメディアへの移動は任意機能である。これらの採用と表示位置、操作時間は製品ごとに決める。

## 再生状態を一つに保つ

各時点で、再生先と再生位置の正本を一つにする。端末内で再生するときは ExoPlayer、Cast 中は受信機の状態に従う。このガイドでは、`MediaSessionService` が端末側の ExoPlayer と MediaSession を所有し、Activity が `MediaController` を通じて操作する構成を前提にする。画面を開き直すたびに別のプレーヤーを作ると、音声の二重再生や再生位置の競合が起きる。

映像を複数の表示領域で見せる場合も、同時に有効な映像出力は一つとする。同じ Activity 内で表示先の親 View を変えるなら、既存の `PlayerView` を移動する。単に大きさを変えるだけなら親から外さず、レイアウトを更新する。Activity が再生成された場合は新しい View を作って再接続してよいが、再生サービスと再生位置は引き継ぐ。Activity や Surface は再生成され得るため、その後に画面を復元する経路を用意する。

折り目をまたぐレイアウトや表示先の移動がある場合は、`TextureView` を使う `PlayerView` を候補にする。描画方式は性能や保護された映像との適合も関わるため、対象コンテンツで確認する。映像は原則 `RESIZE_MODE_FIT` とし、縦横比を保つ。

```xml
<androidx.media3.ui.PlayerView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:surface_type="texture_view"
    app:resize_mode="fit" />
```

端末内で音声を再生するときは、動画に合う `AudioAttributes` を指定し、ExoPlayer の音声フォーカス管理を有効にする。他アプリへのフォーカス移動や出力機器の切断時の振る舞いも決める。動画で音量だけを下げて同時再生を続けない方針なら、その設定と実機での結果を一致させる。Cast 中は受信機と端末から同じ音声を重ねて出さない。

動画の再生開始時に、表示中の動画 Activity のウィンドウへ `FLAG_KEEP_SCREEN_ON` を設定し、画面が自動で暗くなったり消えたりしないようにする。一時停止、再生終了、音声項目への切替、Activity の終了時にはフラグを解除する。動画を再開したら再設定する。画面の再生成や表示先の変更後も、現在の再生状態に合わせて設定と解除を行う。Cast で受信機だけが再生している場合は端末側の点灯維持を解除する。

## 現在のメディアに合わせて映像 UI を切り替える

動画かどうかは、現在の `MediaItem` の MIME 型と、同じメディア ID にひも付けて管理する種別を基に決める。拡張子を除いた表示名、前に再生した項目、画面側の「再生要求中」フラグだけで判定しない。MIME 型が未確定なら動画専用 UI を出す前に、アプリが管理するメタデータや取得結果で確認する。動画ではない項目を選んだら、前の映像枠を速やかに隠す。

`onMediaItemTransition(null)` は「現在項目なし」として処理する。`null` をメタデータ検索や動画判定へ渡さない。項目の切替直後、Controller の通知より先に画面を更新する必要がある場合は、選択した項目の確定済み種別を使い、後から現在項目と照合する。

コンパクト表示は任意であり、画面下部への配置も必須ではない。採用する場合は、その表示領域から詳細表示へ移る入口、戻り先、音声へ切り替えたときの消し方を定義する。最初のフレームが届くまでのシャッターと映像領域は周囲に合う背景色にし、詳細画面では黒へ戻す。起動時に保存済みのタイトルや位置を先に見せても、再生ソースが未接続の `PlayerView` を前回の動画種別だけで表示しない。

Controller の接続に失敗したら、原因と試行回数を記録し、画面が表示中なら再接続する。接続成功時と画面復帰時は、現在項目と映像サイズから表示を同期する。これにより、一時的な未接続後も古い映像枠や操作を残さない。

## 詳細画面には全画面・シーク・Cast・PIP の入口を置く

動画詳細では、再生・一時停止、再生位置を直接動かすシークバー、全画面、Cast、PIP を使えるようにする。指定秒数の前後移動と前後メディアへの移動は任意である。両者は意味が違うため、同じアイコンや操作名を使わない。総時間がまだ分からないメディアでは、長さを前提とするシーク操作を無効にし、長さが確定したら位置と表示を同期する。

埋め込み表示から詳細へ移る場合は、現在の `PlayerView` を詳細用ホストへ移し、`setUseController(true)` と黒い背景を適用する。戻るときは詳細専用の操作を外し、埋め込み表示があれば元のホストへ戻して `setUseController(false)` と背景色を復元する。埋め込み表示がないアプリは、詳細画面だけを表示ホストにすればよい。画面遷移で再生サービスを止めたり、再生位置を初期化したりしない。

Cast の接続先選択には Google Cast Sender SDK の標準 `MediaRouteButton` を使う。PIP は動画詳細で現在項目が動画と確認できる場合に起動する。これらのボタンを `PlayerView` 全面や画面下端の独立した Root オーバーレイへ置くと、コントローラーの自動非表示とずれたり、タイムバーや時刻表示へ重なったりする。採用したコントローラー内へ組み込み、標準操作と同時に表示・非表示を切り替える。使用する Media3 の版で内部レイアウト ID と配置を確認する。

操作部をカスタマイズするときは、通常表示の `PlayerView` 内蔵コントローラーと、姿勢専用表示の操作部を同じ `MediaController` へ接続する。別の ExoPlayer を作らない。姿勢専用の操作部には、`PlayerView` 内のカスタムコントローラーか、`MediaController` を観測・操作する独自 View を使う。`PlayerControlView` を `PlayerView` の外で単体利用する構成は Media3 の公式ガイドで非推奨であり、新規実装の標準形にしない。全画面・PIP・Cast のボタンをそれぞれ十分なタッチ領域（目安 48 dp）で、採用したコントローラーレイアウト（例: `exo_basic_controls`）に配置し、画面ごとに同じ順序で設定する。独自の全画面ボタンを置いたら、Media3 標準の全画面ボタン、狭い幅用の全画面ボタン、不要な overflow 入口を非表示にし、重複した操作を残さない。VR、シャッフル、リピート、字幕、設定なども製品で使わないなら表示を外し、狭い画面で必要な操作が押し出されないようにする。

指定時間の戻し・送りを採用する場合は、中央の再生・一時停止や任意の前後メディア操作と意味を区別する。Media3 1.7.1 のコントローラーレイアウトでは、時間量付きボタンの ID は `exo_rew_with_amount` と `exo_ffwd_with_amount` である。`exo_rew` と `exo_ffwd` だけを探す実装では、このレイアウトのボタンを取得できない。これらの内部 ID は将来も同じとは限らない。採用した Media3 の版と実際に膨らませたレイアウトを確認し、対象 View が見つからない状態を黙って成功扱いにしない。内部レイアウトの変更に追随できない場合は、独自の操作 View に切り替える。`setShowRewindButton(true)` と `setShowFastForwardButton(true)` だけに依存せず、コントローラー生成後とレイアウト切替後に表示、活性状態、透明度、クリック経路を確認する。端末内再生の時間移動は同じ `MediaController.seekBack()` と `seekForward()` へ送り、Cast 中は受信機の再生コントローラーへ送る。

Media3 はコントローラーの表示切替や利用可能コマンドの変更時に子 View の状態を更新する。内蔵コントローラーへ追加したボタンは、Controller 接続後、`PlayerView.setControllerVisibilityListener()` による可視状態変更の通知後、レイアウト再生成後に設定を確認・再適用する。独自 View の場合は自身の可視状態と操作可能状態を同じ契機で更新する。更新処理は現在の再生先と利用可能なコマンドを見て行い、前の表示モードの View を操作しない。`PlayerView` の `controllerAutoShow` を有効にし、Controller 接続後、再生準備完了後、レイアウト適用後、映像領域のタップ時には、表示すべき場面なら `showController()` を呼ぶ。PIP 中や意図して操作を隠した場面では呼ばない。ボタンが存在してもコントローラー全体が非表示なら使えないため、両方の可視状態を確認する。

## 入口や姿勢が変わっても操作の位置関係を保つ

ファイル一覧、通知、埋め込み表示など、どこから動画詳細を開いても、同じ操作は同じ相対位置に置く。通常表示、縦横の切替、全画面、外部表示、半開きでも操作の意味と順序を変えない。採用しない操作は空白だけを残さず取り除く。

開いた内側画面の縦向きと水平な半開きでは、映像領域の下に独立した操作領域を設ける。再生・一時停止と採用した時間移動・前後メディア移動は操作領域の中央、タイムバーはその下、時刻と補助操作は最下段に置く。内蔵コントローラーはこの間表示せず、独立した操作部だけを同じ `MediaController` へつなぐ。通常表示へ戻すときは独立した操作部を隠し、`PlayerView` 内蔵コントローラーを復帰させる。

```text
┌────────────────── 映像（FIT） ──────────────────┐
└───────────────────────────────────────────────┘
┌─────────────── 操作領域 ──────────────────────┐
│            再生／一時停止など                 │
├───────────────── タイムバー ──────────────────┤
│ 現在時刻 / 総時間          Cast  PIP  全画面   │
└───────────────────────────────────────────────┘
```

タイムバーは下部操作列の上、時刻は操作列の開始側、Cast・PIP・全画面は終端側に置く。補助操作の順序は一度決めたら表示モード間でそろえる。中央の再生操作を下部に重複させない。時間移動と前後メディアへの移動を採用するなら、それらも中央にまとめる。映像、タイムバー、時刻と補助操作の上下関係を保ち、狭い領域では任意操作を整理して必須操作や時刻を重ねない。

同じ操作には同じ意味のアイコンと説明を使う。再生と一時停止、全画面にする操作と解除する操作は、状態に合わせてアイコンと `contentDescription` を両方更新する。時間移動には時間が分かるアイコンを使い、メディア単位の前後移動と区別する。非表示の操作をフォーカス順やアクセシビリティツリーに残さない。

## システムバーと折り目を避ける

通常の動画詳細では Window Insets を適用し、タイムバーと下部操作列をシステム UI の内側へ収める。Media3 のコントローラーが `PlayerView` の外枠を基準に配置される構成では、映像コンテンツへの padding だけに頼らず、`PlayerView` の外側のマージンで配置領域そのものを安全領域へ収める。姿勢専用の独自操作部もその外側にマージンを取る。下端だけでなく左右の Insets を反映する。カバー画面を横向きにしたときは、右端のナビゲーション領域が全画面ボタンと重なることがある。Cast 用のシークバーを別に置く場合も同じ安全領域に収める。

全画面でシステムバーを隠すときはその余白を外し、解除時には現在の Insets を再取得して適用する。システムの戻る操作でも通常表示へ復帰できるようにする。内蔵コントローラーと独自操作部の全画面状態を同期する場合は、状態更新で同じコールバックが再度呼ばれても循環しないようにする。全画面ボタンのアイコンと `contentDescription` も移行と解除に合わせて切り替える。画面の回転、ウィンドウのサイズ変更、外部表示の接続・切断後は Insets を再適用する。外部ディスプレイのミラーリングと DeX のウィンドウ表示は別の状態として扱い、表示先だけで没入表示を決めない。映像は全画面でも引き伸ばさず FIT 表示を維持する。

## 上下分割では映像を上、操作を下へ置く

表示先、ウィンドウの向き、折り目の判定、姿勢変更時の配置リセットは [display.md](display.md) に従う。動画画面では、内側画面を開いた縦向きと、内側画面に水平・分離する `HALF_OPENED` の折り目がある場合に、映像を上、操作を下へ置く。開いた内側画面の横向き、カバー画面、外部表示や DeX のウィンドウは、利用可能な領域へ FIT 表示する。山型で横向きのカバー画面を使う構成では上下分割しない。

開いた内側画面の縦向きでは、現在の安全領域と動画の縦横比から、画面全体へ FIT 表示した場合の映像寸法を求める。その寸法を変えずに映像を安全領域の上端へ置き、直下から下端までを独立した操作領域にする。映像を拡大・切り抜きして余白を消さない。余白が足りなければ、操作と重ねず、必要な操作領域を確保できる寸法まで FIT 表示を縮める。

水平の半開きでは、有効な折り目の `bounds` を View の座標へ変換して映像領域の下端を決める。映像を折り目より上、独立した操作領域を下へ置き、遮蔽範囲には操作を置かない。全画面に移っても上下の関係を保つ。通常表示へ戻すときは半開き用の寸法と変換を解除し、内蔵コントローラーと独立した操作部を同時に操作可能にしない。

## PIP は再生サービスへ操作を戻す

PIP へ入るときは、Activity のウィンドウ内で `PlayerView` が占める矩形を取得し、`sourceRectHint` に渡し、映像のアスペクト比も設定する。詳細用コントローラーは PIP 中に隠し、復帰時に戻す。PIP 用に別のプレーヤーは作らない。

追加操作を置くかどうかは製品で決める。置く場合は、再生・一時停止や指定時間移動を既存の再生サービスへ送る。MediaSession からシステムが表示する標準操作との重複や、端末ごとの表示数を確認する。PIP の開始、復帰、アプリを離れた状態での継続、操作結果を実機で確かめる。

## Cast 中は受信機を再生先の正本にする

Cast セッションを開始・復帰するときは、受信機が取得できるメディア URL と MIME 型、タイトル、開始位置を渡す。端末内の `file://` や権限付きの `content://` をそのまま受信機へ渡しても再生できない。認証や短い有効期限を持つメディアは、受信機が利用できる安全な URL を要求ごとに取得する。URL や認証情報をログや永続ストレージへ残さない。

ロード要求を送っただけで成功とみなさない。受信機がロードを受け付けたことを確認してから端末再生を止め、失敗時の端末側の扱いも定める。Cast 中に別の項目を選んだ場合は同じ受信機へ切り替え、意図せず端末側で新たに再生しない。

Cast 中の位置は受信機からの進捗を正本にして保存する。一時停止中の端末プレーヤーが古い位置を保存し、受信機の進捗を上書きしないよう、対象メディアと再生先を識別する。巻き戻しも有効な更新として扱い、値の大小だけで古い情報を判断しない。セッション終了時は最後に確認できた受信機位置を端末へ同期し、その後で Cast 中の保護を解除する。更新間隔や保存先は製品の要件に合わせる。

Cast 中は端末側のローカルタイムバーを隠し、受信機位置を示す Cast 用シークバーだけを同じ場所に表示する。二つのバーを重ねない。Cast 用シークバーにも左右・下端の Window Insets を反映し、シーク操作は受信機へ送る。Cast セッションの開始・終了と再生先の変更時に、表示するバーと現在位置を切り替える。

接続先の選択、ロードの受付、受信機側の再生開始、進捗、切断を別々に記録する。Cast ボタンが表示されたことやロード API の成功だけで、テレビでの再生を検証済みとしない。

## Bluetooth 設定と音声出力先切替の入口を設ける

動画の操作部には、Bluetooth 設定を開くボタンと、システムのメディア出力先画面を開くボタンを設ける。Cast の接続先を選ぶ `MediaRouteButton` は別の操作として扱う。前二者は画面を開く入口であり、押しただけで出力先は変わらない。Cast 用ボタンを端末内の出力先切替ボタンとして流用しない。動画の通常・全画面・半開き表示では両ボタンを利用できるようにし、ミニプレーヤーや音声詳細にも置く製品ではアイコン、順序、`contentDescription` をそろえる。素材は 24 dp 程度でも、各ボタンの操作領域は 48 dp 以上とし、コントローラーの表示・非表示と Insets に連動させる。

Bluetooth 設定への入口には `Settings.ACTION_BLUETOOTH_SETTINGS` を使う。遷移先を確認してから起動し、開けない場合は利用者に知らせる。設定から戻っても再生サービスとプレーヤーを作り直さず、再生状態と UI を同期する。設定画面を開くこと自体には Bluetooth のスキャン権限は不要だが、アプリ内で探索や接続も行う場合は別途権限と実装を検討する。

出力先画面には、前面に表示中の Activity から `MediaRouter2.showSystemOutputSwitcher()` を使う。この引数なし API は Android 14（API 34）以降で利用できる。Android API 拡張 36.1 以降で `showSystemOutputSwitcher(MediaSession.Token)` が使える場合は、再生中のセッションを指定する。Media3 `MediaSession.getPlatformToken()` の値をサービスから UI へ安全に渡し、サービス破棄時には古い値を使わない。別プロセス間で共有する場合は、その構成に合う受け渡し方法を設ける。対象 API がない、トークンを取得できない、またはトークン付き呼び出しが失敗した場合は、利用可能な引数なし API へ戻す。古い Android 版にも対応する製品では、別の入口を設計する。どちらの API も戻り値の `true` は画面の表示を示すだけで、出力先の切替成功を示さない。

AndroidX MediaRouter を代替入口に使う場合は、対象端末でシステム API が要件を満たさないことを確認してから採用する。`MediaRouteButton` と `CATEGORY_LIVE_AUDIO`、出力切替用の `MediaRouterParams`、必要な `MediaTransferReceiver` 登録を、採用するライブラリの版に照らして確認する。この経路は端末や設定によって Cast の選択画面を開くこともあるため、動画の Cast 専用ボタンとは役割を混ぜない。Media3 の platform `MediaSession.Token` を AndroidX MediaRouter の `setMediaSession(Object)` にそのまま渡さない。

DeX ではシステムの出力先画面がアプリの外部ウィンドウではなく端末側へ表示されたり、音量バーが実際の出力経路の値と一致しなかったりすることがある。システム UI の表示先と音量バーをアプリが制御できると前提にしない。外部ウィンドウ上での出力先選択や正しい音量値の表示が製品要件なら、実機でシステム画面の振る舞いを確かめる。アプリ内 UI と `MediaRouter2.transferTo()` を検討する場合も、利用可能なルート、転送結果、音量制御の可否を実機で検証し、失敗時の Bluetooth 設定への入口を残す。

## Manifest と依存関係を機能に合わせて設定する

このガイドの構成には Media3 の ExoPlayer・Session・UI が必要である。折りたたみ姿勢には AndroidX Window、Cast には Google Cast Sender SDK を加える。依存ライブラリの版は対象プロジェクトでそろえる。

バックグラウンド再生用の `MediaSessionService` には `mediaPlayback` の foreground service 種別と必要な権限を宣言する。動画 Activity には PIP の許可を付ける。構成変更を Activity 自身で処理する設計なら、対応する `configChanges` を宣言し、その変更時にレイアウトと Insets を更新する。再生成を許す設計なら、画面の復元を実装する。

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

    <application>
        <meta-data
            android:name="com.google.android.gms.cast.framework.OPTIONS_PROVIDER_CLASS_NAME"
            android:value="com.example.app.CastOptionsProvider" />

        <activity
            android:name=".VideoActivity"
            android:supportsPictureInPicture="true"
            android:configChanges="orientation|screenSize|smallestScreenSize|screenLayout" />

        <service
            android:name=".PlaybackService"
            android:foregroundServiceType="mediaPlayback"
            android:exported="true">
            <intent-filter>
                <action android:name="androidx.media3.session.MediaSessionService" />
            </intent-filter>
        </service>
    </application>
</manifest>
```

`com.example.app` は対象アプリのパッケージ名に置き換える。Cast の `OptionsProvider` は受信機の設定に合わせて実装する。`MediaRouteButton` を置く画面では AppCompat / MediaRouter のテーマ属性、とくに `colorPrimary` とボタンの色を確認する。暗い動画画面でも Cast アイコンが見える配色にする。Manifest とテーマは対象 SDK と使用ライブラリの公式文書に照らして確認する。

## 実機で確認する

ビルド、lint、インストール、Activity 起動、画面操作は別の確認結果として記録する。端末のシリアルを明示し、起動には対象 Activity を指定する。`monkey` や利用者設定の変更は行わない。

- **再生と切替**: 音声と動画の切替、現在項目の消去、Controller 再接続で表示と再生位置が一致する。音声フォーカス移動や出力機器の切断時にも二重再生しない。
- **画面の点灯**: 動画再生開始時に動画 Activity の `FLAG_KEEP_SCREEN_ON` が設定され、再生中に画面が自動で暗くならない。一時停止、再生終了、音声への切替、Activity の終了、Cast への切替後はフラグが解除される。再開、画面回転、表示先変更後も状態を確認する。
- **画面遷移**: 詳細の開始・終了、任意の埋め込み表示との往復、回転、サイズ変更後も再生が続き、黒い空映像や古い操作が残らない。
- **操作**: シーク、全画面切替と解除、システム戻る、アイコンと読み上げ文言、タッチ・マウス・キーボードの操作を確認する。
- **コントローラーの配置と表示**: 開いた縦画面を含む各構成で、採用した時間移動・Cast・PIP・全画面が見え、時刻・タイムバー・システム UI と重ならないことを確認する。コントローラー生成直後、Controller 接続後、再生準備完了後、姿勢変更後、映像領域タップ後も表示と活性状態を調べる。標準操作との重複や、画面外へ押し出されたボタンがないことも確かめる。
- **PIP**: 開始・復帰・再生継続と追加操作が、同じ再生サービスへ届く。
- **Cast**: 実際の受信機で接続、再生、項目切替、シーク、進捗保存、切断後の位置を確認する。認証付き URL の場合は失効と再取得も試す。
- **折りたたみと外部表示**: 表示先と姿勢の確認は [display.md](display.md) に従う。開いた内側画面の縦向きでは、映像が安全領域の上端にあり、直下から下端までの独立した操作領域に中央操作、タイムバー、時刻、補助操作が収まることを確認する。水平な半開きでも映像が上・操作が下の同じ関係になる。内側画面の横向きと山型で使う横向きのカバー画面では上下分割しない。姿勢を切り替えても古い映像寸法が残らず、操作が折り目やシステム UI と重ならないことを目視する。
- **Bluetooth 設定と音声出力先切替**: 動画の通常・全画面・半開き表示で両ボタンを操作し、Bluetooth 設定、システムの出力先画面、Cast の各ボタンが目的の画面を開くか確認する。端末スピーカーと利用する Bluetooth 機器で、候補表示、選択後の実際の音、出力先画面の表示位置、音量、設定から戻った後の再生位置を確認する。必要なら選択ルートとその音量、`RoutingController` の音量、`AudioManager.STREAM_MUSIC` の値を同時に記録する。画面表示の成功や Bluetooth 接続表示だけで転送成功とみなさない。機器切断時の自動一時停止を求める製品では、`ACTION_AUDIO_BECOMING_NOISY` とルート変更監視を別途検証する。

特定の端末でしか見られない挙動は、その端末と OS の版、画面状態、再現手順、再生先、実画面の観察結果を添えて対象プロジェクトの課題管理先で扱う。未確認の場面をビルド成功から推測して「対応済み」としない。

操作が見えない、押せない、重なる場合は、通常表示と姿勢専用表示のどちらか、各ボタン・タイムバー・操作列・Cast 用シークバーの可視状態、活性状態、画面座標を記録する。個別アイコンだけでなくコントローラー全体の状態も見る。スクリーンショットと診断ログを同じ操作時点で取得し、複数ディスプレイ接続時は対象 Activity の display ID と撮影した画面が一致していることを確認する。

## 用語

- **`MediaSessionService`**: 再生状態と MediaSession を持ち、画面外でも再生を続けるためのサービス。
- **`MediaController`**: 画面から MediaSession へ接続し、状態を読み操作を送る窓口。
- **`PlayerView`**: Media3 の映像と標準コントローラーを表示する View。
- **`TextureView`**: View の移動や変形を伴う構成で使える映像出力方式の一つ。
- **Window Insets**: システムバーなどが占める領域をレイアウトへ伝える情報。
- **`FoldingFeature`**: 折り目の状態、向き、位置を取得する WindowManager の情報。表示判定は [display.md](display.md) を参照する。
- **PIP**: アプリの外でも小さな動画を表示する Android の画面モード。
- **Cast**: 受信機がメディアを取得し、受信機側で再生する仕組み。

実装時は、使用する Media3、WindowManager、Android SDK、Cast Sender SDK の版に合う公式 API 文書も確認する。このガイドで判断できない仕様は、対象プロジェクトの要件と実機の再現結果に戻って決める。

参照先: [WindowInfoTracker の API](https://developer.android.com/reference/androidx/window/layout/WindowInfoTracker)、[WindowManager Extensions の状態定義](https://source.android.com/docs/core/display/windowmanager-extensions)、[Media3 の PlayerView API](https://developer.android.com/reference/androidx/media3/ui/PlayerView)、[Media3 の UI 構成](https://developer.android.com/media/media3/ui/playerview)、[Media3 のバックグラウンド再生](https://developer.android.com/media/media3/session/background-playback)、[PIP の実装](https://developer.android.com/develop/ui/views/picture-in-picture)、[折りたたみ端末への対応](https://developer.android.com/develop/adaptive-apps/guides/foldables/make-your-app-fold-aware)、[MediaRouter2 の API](https://developer.android.com/reference/android/media/MediaRouter2)、[Bluetooth 設定の Intent](https://developer.android.com/reference/android/provider/Settings#ACTION_BLUETOOTH_SETTINGS)、[Google Cast Sender の統合](https://developers.google.com/cast/docs/android_sender/integrate)。
