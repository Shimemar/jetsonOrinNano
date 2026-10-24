# Jetson Orin Nano Super モード対応記録

作業日：2026-10-24

## 対象機器

| 項目 | 内容 |
|---|---|
| ボード | NVIDIA Jetson Orin Nano Developer Kit（モジュール p3767-0003 / 8GB、キャリアボード p3768-0000） |
| OS | Jetson Linux（L4T）R39.2.1 / Ubuntu 24.04.4 LTS |
| ストレージ | NVMe（`/dev/nvme0n1p1`） |
| 接続 | USB 接続のデバイスモードネットワーク `192.168.55.1`（SSH ユーザー `shimizu`） |

## 1. 対応前の状態

| 項目 | 値 |
|---|---|
| モデル表示 | NVIDIA Jetson Orin Nano Developer Kit |
| 電力モード | 15W（ID=0） |
| 選べるモード | 15W / 7W のみ（25W / MAXN SUPER なし） |
| `/etc/nvpmodel.conf` のリンク先 | `/etc/nvpmodel/nvpmodel_p3767_0003.conf` |
| 起動時の DTB | `kernel_tegra234-p3768-0000+p3767-0003-nv.dtb`（Super 版ではない） |
| CPU 上限 | 1510.4 MHz |
| GPU 上限 | 624.75 MHz |
| メモリ（EMC）上限 | 2133 MHz |
| `jetson_clocks` | 無効 |

## 2. 原因

ボードが Super 版ではない通常の構成（`jetson-orin-nano-devkit`）で書き込まれていた。

- 起動時に `/etc/systemd/nvpower.sh` が DTB の compatible 文字列を見て、`/etc/nvpmodel.conf` のリンク先を決める。文字列に `-super` が含まれていれば `_super.conf` が選ばれる。
- 通常版の DTB で起動していたため、15W / 7W しかない `nvpmodel_p3767_0003.conf` が選ばれていた。
- Super 版の設定（`nvpmodel_p3767_0003_super.conf`：15W / 25W / MAXN_SUPER）と Super 版の DTB は、ファイルとしては最初から入っていた。

## 3. 試したこと：カーネル DTB だけを切り替える（効果なし）

ボードの書き込みはせずに、Jetson 上の設定だけで Super モードにできるか試した。

1. `/boot/extlinux/extlinux.conf` を `extlinux.conf.orig` として退避した。
2. `primary` エントリに `FDT /boot/tegra234-p3768-0000+p3767-0003-nv-super.dtb` を追加した。
3. 起動できなかったときのために、元の DTB で起動する `original` エントリを追加した。
4. 再起動した。

**結果**

- モデル表示は「… Developer Kit Super」になり、`/etc/nvpmodel.conf` も `_super.conf` に自動で切り替わった。`nvpmodel -m 2` で MAXN_SUPER にも設定できた。
- しかし実際のクロックは上がらなかった（GPU 624.75 MHz のまま、CPU は 1497.6 MHz に少し下がった）。
- 周波数の上限は、QSPI に入っている電源管理ファームウェア（BPMP）の設定で決まっていて、そこは通常版のままだった（`/sys/kernel/debug/bpmp/debug/clk/nafll_gpc0/max_rate` = 624750000）。

**結論**：Super の性能を出すには、ホスト PC から Super 構成で書き込み直し、QSPI 側の設定も更新する必要がある。

## 4. 対応：ホスト PC から Super 構成で書き込み直す

公式の [Jetson Linux R39.2 Quick Start](https://docs.nvidia.com/jetson/archives/r39.2/DeveloperGuide/IN/QuickStart.html) の手順に沿って行った。

**注意**：`--erase-all` を付けると NVMe の内容はすべて消える。必要なデータは先に退避しておくこと。

### 4.1 ホスト PC の準備

- Ubuntu 22.04 または 20.04（x86_64）
- Jetson の USB-C ポートとホスト PC を USB ケーブル（データ通信対応）で接続する

```bash
mkdir -p ~/jetson_r39.2.1 && cd ~/jetson_r39.2.1
# https://developer.nvidia.com/linux-tegra から R39.2.1 の BSP と Sample Root Filesystem を入手する
tar xf Jetson_Linux_R39.2.1_aarch64.tbz2
sudo tar xpf Tegra_Linux_Sample-Root-Filesystem_R39.2.1_aarch64.tbz2 -C Linux_for_Tegra/rootfs/
cd Linux_for_Tegra/
sudo ./tools/l4t_flash_prerequisites.sh
sudo ./apply_binaries.sh
ls jetson-orin-nano-devkit-super.conf   # Super 用の構成ファイルがあることを確認する
```

### 4.2 リカバリモードにする

1. 電源ケーブルを抜く。
2. 12 ピンのボタン用ヘッダーで、REC（FC REC）と GND をジャンパーでショートする。
3. 電源ケーブルを差す。
4. ホスト PC で `lsusb | grep -i nvidia` を実行し、`ID 0955:7523 Nvidia Corp.`（Orin Nano 8GB）が表示されることを確認する。
5. 確認できたらジャンパーを外す。

### 4.3 書き込み

```bash
sudo ./tools/kernel_flash/l4t_initrd_flash.sh --showlogs --erase-all \
  jetson-orin-nano-devkit-super internal
```

QSPI（ブートローダーと BPMP ファームウェア）と NVMe の両方に書き込まれる。書き込みは成功した。

## 5. 対応後の確認

```bash
tr -d '\0' < /proc/device-tree/model
ls -l /etc/nvpmodel.conf
sudo nvpmodel -q
sudo jetson_clocks --show | grep -E "cpu0|GPU|EMC"
sudo cat /sys/kernel/debug/bpmp/debug/clk/nafll_gpc0/max_rate
```

| 項目 | 対応前 | 対応後 |
|---|---|---|
| モデル表示 | Orin Nano Developer Kit | Orin Nano Engineering Reference Developer Kit **Super** |
| `/etc/nvpmodel.conf` のリンク先 | `nvpmodel_p3767_0003.conf` | `nvpmodel_p3767_0003_super.conf` |
| 選べるモード | 15W / 7W | 15W / 25W / MAXN_SUPER |
| 現在のモード | 15W | **MAXN_SUPER（ID=2）** |
| CPU 上限 | 1510.4 MHz | **1728 MHz** |
| GPU 上限 | 624.75 MHz | **1020 MHz**（BPMP 側の上限も 1020 MHz） |
| メモリ（EMC）上限 | 2133 MHz | **3199 MHz** |

### GPU の稼働 TPC 数について

- 対応後は `Active GPU TPCs: 4` と表示される（対応前は 8 と表示されていた）。
- これは正しい値と判断した。Orin Nano 8GB の GPU は 1024 CUDA コアで、1 TPC = 2 SM = 256 コアなので 4 TPC になる。
- 設定ファイルでも、15W / 25W / MAXN_SUPER のすべてで `TPC_PG_MASK=240`（4 TPC）が指定されている。
- 対応前の 8 は、このマスクが反映されていなかったときの表示と考えられる。
- 実際の GPU で SM 数を数えてはいない。確かめるには `deviceQuery` で SM 数が 8 であることを確認する。

## 6. 運用メモ

| 操作 | コマンド |
|---|---|
| 現在のモードを確認する | `sudo nvpmodel -q` |
| 15W にする | `sudo nvpmodel -m 0` |
| 25W にする | `sudo nvpmodel -m 1` |
| MAXN_SUPER にする | `sudo nvpmodel -m 2` |
| クロックを最大に固定する（性能測定の前など） | `sudo jetson_clocks` |
| クロックの状態を見る | `sudo jetson_clocks --show` |

- モードによっては、切り替えのときに再起動を求められる（`yes` と入力すると再起動する）。
- MAXN_SUPER は消費電力が大きいので、19V のACアダプタを使い、ファンが動いていることを確認する。
- 3 章で変更した `extlinux.conf` は、書き込み直したときに上書きされたので、元に戻す作業は不要。

## 参考

- [NVIDIA Jetson Linux R39.2 Quick Start](https://docs.nvidia.com/jetson/archives/r39.2/DeveloperGuide/IN/QuickStart.html)
- [JetPack 7.2 / Jetson Linux r39.2 on Jetson Orin Nano Developer Kit — NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/jetpack-7-2-jetson-linux-r39-2-on-jetson-orin-nano-developer-kit-getting-started-and-feedback-thread/372151?page=2)
