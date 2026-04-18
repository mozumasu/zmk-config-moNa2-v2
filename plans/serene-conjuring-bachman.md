# ポインティングデバイスのモジュール化計画

## Context

moNa2-v2 は Seeeduino XIAO BLE (nRF52840) を使用した分割無線キーボード。
現在、右側に PMW3610 トラックボール (SPI接続) が固定実装されている。

**目標**: トラックボールとトラックパッド (IQS7211E) を磁石で着脱可能なモジュールにする。

**スコープ**: 右側のみ。Phase 1 として排他的モジュール交換 (トラックボール or トラックパッド) を実装する。
同時使用は PCB 改版後の Phase 2 として将来対応。

**制約**: XIAO BLE のヘッダーピン (D0-D10) は全て使用済み。追加 GPIO パッドなし。
SPI (トラックボール) と I2C (トラックパッド) で同じ物理ピンを共有し、ファームウェアのスニペットで切り替える。

---

## アーキテクチャ: ZMK Snippets 方式

LisM キーボードのパターンに倣い、**ZMK Snippets** でポインティングデバイスを切り替える。

**選定理由**:

- ベースシールドはポインティングデバイスに依存しない
- スニペットの組み合わせで trackball / trackpad を切り替え可能（将来的に両方同時対応も拡張可能）
- SO-SHO方式 (シールド分離) のようなシールド定義の重複が不要

---

## フェーズ1: 排他的モジュール交換 (PCB改版不要)

トラックボール **または** トラックパッドを磁石で付け替える。同じピン (D4/D5/D0) を共有し、ファームウェアで使用デバイスを切り替える。

### 磁石コネクタのピンアサイン (6ピン)

| Pin | Signal | トラックボール | トラックパッド |
|-----|--------|----------------|----------------|
| 1 | VCC (3.3V) | VCC | VCC |
| 2 | GND | GND | GND |
| 3 | D4 (P0.04) | SPI SDIO | I2C SDA |
| 4 | D5 (P0.05) | SPI SCK | I2C SCL |
| 5 | D0 (P0.02) | IRQ (MOTION) | IRQ (RDY) |
| 6 | P0.09 | SPI CS | power-gpios (省電力制御) |

### ファームウェア変更

#### 1. `config/west.yml` - IQS7211E ドライバ追加

```yaml
remotes:
  # 既存に追加
  - name: sekigon-gonnoc
    url-base: https://github.com/sekigon-gonnoc

projects:
  # 既存に追加
  - name: zmk-driver-iqs7211e
    remote: sekigon-gonnoc
    revision: main
```

#### 2. `boards/shields/mona2/mona2_r.overlay` - ポインティングデバイス設定を除去

SPI0, PMW3610, pinctrl セクションを全て削除し、キーマトリクスのみ残す:

```dts
#include "mona2.dtsi"

&default_transform {
    col-offset = <6>;
};

&kscan0 {
    col-gpios
        = <&xiao_d 10 GPIO_ACTIVE_HIGH>
        , <&xiao_d 9 GPIO_ACTIVE_HIGH>
        , <&xiao_d 8 GPIO_ACTIVE_HIGH>
        , <&xiao_d 7 GPIO_ACTIVE_HIGH>
        , <&gpio0 10 GPIO_ACTIVE_HIGH>
        ;
};
```

#### 3. `boards/shields/mona2/mona2.dtsi` - trackball_central_listener を削除

`trackball_central_listener` ノード (L89-97) を削除。各スニペットが独自のリスナーを定義する。

#### 4. `boards/shields/mona2/Kconfig.defconfig` - ポインティング関連デフォルトを除去

`SHIELD_MONA2_R` ブロックから以下を削除:

- `CONFIG_INPUT`, `CONFIG_ZMK_POINTING`, `CONFIG_SPI`, `CONFIG_PMW3610`

#### 5. 新規: `snippets/trackball/` ディレクトリ

**`snippets/trackball/snippet.yml`**:

```yaml
name: trackball
append:
  EXTRA_DTC_OVERLAY_FILE: trackball.overlay
  EXTRA_CONF_FILE: trackball.conf
```

**`snippets/trackball/trackball.overlay`**:
現在の `mona2_r.overlay` から抽出した pinctrl + SPI0 + PMW3610 + input-listener 定義。

**`snippets/trackball/trackball.conf`**:

```conf
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_ZMK_POINTING=y
CONFIG_PMW3610=y
```

#### 6. 新規: `snippets/trackpad/` ディレクトリ

**`snippets/trackpad/snippet.yml`**:

```yaml
name: trackpad
append:
  EXTRA_DTC_OVERLAY_FILE: trackpad.overlay
  EXTRA_CONF_FILE: trackpad.conf
```

**`snippets/trackpad/trackpad.overlay`**:
I2C1 (TWIM1) の pinctrl + IQS7211E デバイスノード + input-listener 定義。

- I2C SDA: P0.04, SCL: P0.05 (SPI と同じ物理ピンを再利用)
- IRQ: P0.02 (トラックボールと同じピンを再利用)

**`snippets/trackpad/trackpad.conf`**:

```conf
CONFIG_I2C=y
CONFIG_INPUT=y
CONFIG_ZMK_POINTING=y
CONFIG_IQS7211E=y
```

#### 7. `build.yaml` - ビルドバリアント追加

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: mona2_r rgbled_adapter
    snippet: trackball studio-rpc-usb-uart
    artifact-name: mona2_r-trackball
  - board: seeeduino_xiao_ble
    shield: mona2_r rgbled_adapter
    snippet: trackpad studio-rpc-usb-uart
    artifact-name: mona2_r-trackpad
  - board: seeeduino_xiao_ble
    shield: mona2_l rgbled_adapter
  - board: seeeduino_xiao_ble
    shield: settings_reset
```

#### 8. `zephyr/module.yml` - snippet_root 追加

---

## (将来) フェーズ2: 同時使用 (PCB改版が必要)

> Phase 1 完了後、PCB 改版を行った場合に対応。今回のスコープ外。

- PCB に I2C1 用の追加 GPIO パッド (SDA, SCL, IRQ) を引き出す
- `snippets/trackball-and-trackpad/` スニペットで SPI0 + I2C1 を同時稼働
- 2つの input-listener が並列動作

---

## 修正対象ファイル一覧

| ファイル | 操作 |
|---------|------|
| `config/west.yml` | 編集: sekigon-gonnoc remote + zmk-driver-iqs7211e 追加 |
| `boards/shields/mona2/mona2_r.overlay` | 編集: SPI/PMW3610/listener を除去 |
| `boards/shields/mona2/mona2.dtsi` | 編集: trackball_central_listener を除去 |
| `boards/shields/mona2/Kconfig.defconfig` | 編集: ポインティング関連 Kconfig を除去 |
| `snippets/trackball/snippet.yml` | 新規作成 |
| `snippets/trackball/trackball.overlay` | 新規作成 |
| `snippets/trackball/trackball.conf` | 新規作成 |
| `snippets/trackpad/snippet.yml` | 新規作成 |
| `snippets/trackpad/trackpad.overlay` | 新規作成 |
| `snippets/trackpad/trackpad.conf` | 新規作成 |
| `build.yaml` | 編集: スニペット付きビルドバリアント追加 |
| `zephyr/module.yml` | 編集: snippet_root 追加 |

---

## 検証方法

1. **トラックボールスニペットのビルドテスト**: GitHub Actions でビルドが通ることを確認 (既存動作と同一ファームウェアが生成されるはず)
2. **トラックパッドスニペットのビルドテスト**: GitHub Actions でビルドが通ることを確認
3. **実機テスト (トラックボール)**: 現在のトラックボールモジュールで動作確認 (マウスカーソル移動、スクロール、クリック)
4. **実機テスト (トラックパッド)**: IQS7211E トラックパッドモジュールで動作確認
5. **磁石着脱テスト**: モジュール交換後にファームウェアを書き換えて正常動作を確認
