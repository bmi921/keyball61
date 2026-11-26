# keyball61 
Keyball61(トラックボール搭載の自作分割キーボード)で、AliExpress等で購入した
RP2040ベースのPro Micro互換コントローラーを使っている場合に発生する、
トラックボール操作時の一方向ドリフトを修正するためのファームウェアパッチです。

公式リポジトリ [Yowkees/keyball](https://github.com/Yowkees/keyball) の
`qmk_firmware/keyboards/keyball/` 以下にあるファイルへの差分として使う想定です。

## 症状

トラックボールを指で固定して円を描くように回転させると、カーソル移動やスクロールが
円運動に加えて一定方向に少しずつドリフトしていく。

円を描く向きを逆にするとドリフトの方向も反転する。CPIやスクロール分周などの
ソフトウェア設定ではなく、**PMW3360センサーチップが基板上でわずかに回転して
実装されていること(光学的な軸ズレ)** が原因と考えられる。

これはAliExpressなどで販売されている互換トラックボールモジュール/コントローラー
基板でよく見られる個体差で、公式のKeyball本体設計自体の問題ではない。

## 原因箇所

`keyboards/keyball/lib/keyball/keyball.c` の `pointing_device_driver_get_report()` が、
センサーから読み取った生の移動量 (`pmw3360_motion_burst()` の結果) を
`keyball.this_motion` / `keyball.that_motion` に蓄積している。

通常のカーソル移動 (`keyball_on_apply_motion_to_mouse_move`) とスクロール
(`keyball_on_apply_motion_to_mouse_scroll`) は、どちらもこの蓄積された生の
移動量を後段で消費するだけなので、**この2つを個別に直すのではなく、
生データが読み取られた直後の1箇所を直せば両方に効く**。

## 修正内容

`keyball.c` の `pmw3360_motion_burst()` 呼び出し直後、`this_motion` に加算する
直前に、固定小数点(Q10, ×1024)の2次元回転行列を適用して、センサーの
物理的な取り付け角度のズレを打ち消す。

```c
r->x' = (x*cos - y*sin) >> 10
r->y' = (x*sin + y*cos) >> 10
```

- cos/sin の計算(浮動小数点)は起動時に1回だけ実行し、以降はホットパス
  (センサーレポートのたびに呼ばれる箇所) では整数演算のみ行う
- AVR(Pro Micro)/RP2040どちらでも軽量に動作する
- トラックボールを持つ側(`this_have_ball`)でのみ動作するコードパス内に
  実装しているため、分割キーボードの左右どちら側にボールがあっても
  自動的に正しい側だけに適用される(`is_left`による分岐は不要)
- スクロールスナップの除算・余り処理など、既存のスクロール変換ロジックには
  一切手を加えていない

角度と回転方向は、キーマップの `config.h` の `#define` だけで調整できる:

```c
#define KEYBALL_ROTATE_ANGLE 5.0  // 度。センサーの物理ズレを打ち消す角度
#define KEYBALL_ROTATE_DIR   1    // +1 か -1。回転方向を反転する
```

`keyboards/keyball/keyball61/keymaps/via/config.h` に設定例あり。

## このリポジトリの構成

Yowkees/keyball のファイル構成と同じパスに、変更したファイルだけを置いている。

```
keyboards/keyball/
├── lib/keyball/keyball.c                    # 回転補正の本体
├── drivers/pmw3360/pmw3360.c                # RP2040ビルド時のSPI分周バグ修正(下記)
└── keyball61/keymaps/via/config.h           # 角度設定の例
```

使い方: `qmk_firmware/keyboards/keyball/` 以下の同名ファイルをこのリポジトリの
内容で置き換える(または diff を当てる)。

## 副次的に見つかった修正: RP2040ビルド時のSPI分周バグ

`pmw3360.c` の SPI クロック分周計算が `F_CPU`(AVR専用マクロ)に依存しており、
RP2040(ChibiOSプラットフォーム)ではビルドできない:

```c
// 修正前(AVR専用)
#define PMW3360_SPI_DIVISOR (F_CPU / PMW3360_CLOCKS)

// 修正後
#ifdef __AVR__
#    define PMW3360_SPI_DIVISOR (F_CPU / PMW3360_CLOCKS)
#else
#    define PMW3360_SPI_DIVISOR 64  // QMK公式 drivers/sensors/pmw33xx_common.h と同じ値
#endif
```

QMK公式の `drivers/sensors/pmw33xx_common.h` が同じ分岐(AVR以外は固定値64)を
採用しているため、それに合わせた。

## 角度の調整方法

1. まず `KEYBALL_ROTATE_ANGLE 5.0` でビルド・書き込みし、円運動でドリフトの
   残り具合を確認する
2. ドリフトが残る場合、角度を大きく/小さく振って二分探索的に追い込む
   (逆方向にズレる場合は `KEYBALL_ROTATE_DIR` を `-1` にして符号を確認)
3. カーソル移動・スクロールの両方で改善を確認する

## 既知の未解決事項: ビルド環境

このリポジトリはソースパッチのみを含み、実際にRP2040向けにビルドが通る
環境構築までは完了していない。試行の過程で分かったこと:

- Yowkees/keyball 公式CIは QMK `0.22.14` に固定してビルドしている
  (`.github/workflows/build-all.yml` 参照。最新のQMK本体ではAPIの非互換で
  ビルドできない)
- `CONVERT_TO=promicro_rp2040` はQMK 0.22.14時点ではSparkFun Pro Micro RP2040
  向けの変換ターゲットとして直接使える(最新QMKでは `sparkfun_pm2040` への
  非推奨エイリアスになっている)
- duplexmatrix(独自マトリクススキャン)自体はQMKの移植性のあるGPIO APIを
  使っており、RP2040非互換ではない。過去に報告されていた非互換の疑い
  ([Yowkees/keyball#572](https://github.com/Yowkees/keyball/issues/572)) は、
  実際にはビルドに使うQMK本体のバージョン不一致が原因だった可能性が高い
- 上記のSPI分周バグを直した先、RP2040向けのSPIピン定義
  (`SPI_SCK_PIN`/`SPI_MOSI_PIN`/`SPI_MISO_PIN`/`SPI_DRIVER`)がSTM32基板
  向けのデフォルト値のままで、RP2040では未定義になる問題が残っている
  (未修正)

実機のUF2が実際どのソース・どのビルド設定から生成されたものかは特定できて
いない。ビルドを再開する場合は、ボードの購入元やコミュニティで実際に
使われているビルド設定を確認するところから始めるのが早い。
