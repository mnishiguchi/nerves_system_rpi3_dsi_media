# Raspberry Pi 3 向け DSI 表示対応 Nerves システムのカスタマイズ概要

## 概要

この文書は、`nerves_system_rpi3` をもとにした独自 Nerves システムの主なカスタマイズ内容をまとめる。

目的は、Raspberry Pi 3 で DSI 接続の表示装置を使い、Scenic による画面表示、RTSP カメラ映像表示、MP4 再生、DRM/KMS 検証を行えるようにすることである。

## 基本方針

hardware、boot、kernel、映像処理に関わるコマンド群は Nerves システム側に置く。

一方で、表示内容、camera stream の扱い、画面構成などは Elixir/Scenic アプリ側に置く。

## 主な関連ファイル

```text
VERSION
linux-6.12.defconfig
nerves_defconfig
fwup.conf.eex
fwup.conf
Config.in
```

## DSI overlay files

ファームウェアイメージの boot partition に、DSI 表示用の overlay files を含める。

```text
overlays/vc4-kms-dsi-7inch.dtbo
overlays/vc4-kms-dsi-ili9881-5inch.dtbo
overlays/vc4-kms-dsi-ili9881-7inch.dtbo
```

これにより、`config.txt` で使用する表示装置に対応した overlay を指定できる。

複数の overlay files を含めているが、実際に有効になる表示装置は `config.txt` で指定した overlay によって決まる。

## kernel 設定

DSI/DRM 表示系を起動時の早い段階で使えるようにするため、主要な表示関連機能を module ではなく kernel 組み込み寄りにする。

主な設定は次の通り。

```text
CONFIG_DRM=y
CONFIG_DRM_VC4=y
CONFIG_DRM_PANEL_SIMPLE=y
CONFIG_DRM_TOSHIBA_TC358762=y
CONFIG_DRM_FBDEV_EMULATION=y
CONFIG_FRAMEBUFFER_CONSOLE=y
CONFIG_BACKLIGHT_CLASS_DEVICE=y
CONFIG_BACKLIGHT_RPI=y
CONFIG_REGULATOR_RASPBERRYPI_TOUCHSCREEN_ATTINY=y
```

これにより、DSI 表示系が module の読み込み順に依存しにくくなる。

## FFmpeg

RTSP カメラ映像の表示、MP4 再生、録画時間の取得のために FFmpeg を含める。

```text
BR2_PACKAGE_FFMPEG=y
BR2_PACKAGE_FFMPEG_FFMPEG=y
BR2_PACKAGE_FFMPEG_FFPROBE=y
BR2_PACKAGE_FFMPEG_SWSCALE=y
```

用途は次の通り。

- `ffmpeg` による RTSP カメラ映像表示
- `ffmpeg` による MP4 再生
- `ffprobe` による録画時間取得
- `swscale` による映像の拡大縮小

## GStreamer

DRM overlay plane の検証用に GStreamer を含める。

```text
BR2_PACKAGE_GSTREAMER1=y
BR2_PACKAGE_GSTREAMER1_INSTALL_TOOLS=y
BR2_PACKAGE_GST1_PLUGINS_BASE=y
BR2_PACKAGE_GST1_PLUGINS_GOOD=y
BR2_PACKAGE_GST1_PLUGINS_BAD=y
BR2_PACKAGE_GST1_PLUGINS_BAD_PLUGIN_KMS=y
BR2_PACKAGE_GST1_PLUGINS_GOOD_PLUGIN_V4L2=y
BR2_PACKAGE_GST1_PLUGINS_GOOD_PLUGIN_V4L2_PROBE=y
```

想定している検証経路は次の通り。

```text
rtspsrc → rtph264depay → h264parse → v4l2h264dec → videoconvert → kmssink
```

`V4L2_PROBE` は、`v4l2h264dec` などの decoder elements を GStreamer から見えるようにするために必要である。

## software H.264 decode

Pi 3 の `v4l2h264dec` 経路では、映像右端が欠ける問題がある。その回避策として `gst-libav` を含め、`avdec_h264` を使えるようにする。

```text
BR2_PACKAGE_GST1_LIBAV=y
```

現在の使い分けは次の通り。

- SINGLE view では `avdec_h264` を使う
- QUAD view では既存の FFmpeg + cairo 経路を使う

## DRM/KMS 診断

対象機器上で DRM/KMS の状態を確認するため、libdrm test tools を含める。

```text
BR2_PACKAGE_LIBDRM_INSTALL_TESTS=y
```

主に次のコマンドを使う。

```text
modetest
drm_info
```

## 確認方法

build 時に確認する。

```sh
mix deps.get
mix compile
mix nerves.system.lint nerves_defconfig
mix generate_fwup_conf

grep -n "vc4-kms-dsi" fwup.conf.eex fwup.conf
grep -n "CONFIG_DRM=y" linux-6.12.defconfig
grep -n "CONFIG_DRM_VC4=y" linux-6.12.defconfig
grep -n "BR2_PACKAGE_FFMPEG" nerves_defconfig
grep -n "BR2_PACKAGE_GST1" nerves_defconfig
```

対象機器上で確認する。

```sh
ls -l /dev/dri
modetest -M vc4
drm_info

ffmpeg -version
ffprobe -version

gst-launch-1.0 --version
gst-inspect-1.0 kmssink
gst-inspect-1.0 v4l2h264dec
gst-inspect-1.0 avdec_h264
```

アプリ側では、次を確認する。

- DSI 表示装置が起動時に点灯する
- Scenic が画面を描画できる
- RTSP カメラ映像を表示できる
- `avdec_h264` 使用時に映像右端が欠けない
- MP4 再生と録画時間取得が動作する

## 運用上の注意

`fwup.conf` は `fwup.conf.eex` から生成される。

boot partition に含める files を変更した場合は、`fwup.conf.eex` を編集し、次を実行する。

```sh
mix generate_fwup_conf
```

上流の `nerves_system_rpi3` に追従する場合は、DSI overlay files、kernel 設定、FFmpeg、GStreamer、`gst-libav`、libdrm test tools が引き続き有効か確認する。
