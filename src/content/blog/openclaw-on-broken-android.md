---
title: '画面バキバキのAndroidにAIエージェントを構築した話【Mac Mini不要】'
description: '画面に縦線が走るジャンクのRedMagic 8にOpenClawを入れて、WhatsApp経由AIエージェントを24/7運用する方法。Mac Mini 7万円の代替を0円で。'
category: 'Tech'
pubDate: '2026-02-12'
heroImage: ''
---

# Mac Mini並べてる場合じゃない

最近、Mac Miniを大量に並べてAIエージェントを走らせてる動画がバズってた。見栄えはいい。映える。だが冷静に考えてほしい。

**OpenClawがやってることはAPIを叩いてるだけだ。**

Node.jsが動いて、インターネットに繋がれば、それでいい。7万円のMac Miniの性能の1%も使ってない。

じゃあ何で動かすか？**ジャンクのAndroidでいい。**

## 実際にやってみた

手元にあったのは **RedMagic 8**。画面に縦線が走ってる。普通なら文鎮。ゴミ。

- **SoC**: Snapdragon 8 Gen 2
- **RAM**: 12GB
- **状態**: 画面に縦線（ギリギリ読めるレベル）
- **価値**: 0円

これを**WhatsApp経由で会話できるAIエージェント**に仕立てた。

## 必要なもの

- 画面壊れたAndroid（ADBが使える状態）
- USB-Cケーブル
- WiFi環境
- OpenClawアカウント
- LLM APIキー（OpenAI, Anthropic等）

## 手順

### Step 1: ADB接続

PCにplatform-toolsを入れる。Android Studioは要らない。100MBもない。

```bash
# Windows
winget install Google.PlatformTools

# 接続確認
adb devices
```

USBデバッグが事前にONになってることが前提。画面が完全に死んでると初期設定できないので注意。

### Step 2: Termuxインストール

Play Store版は更新止まってるので **F-Droid版** 一択。

```bash
# APKをダウンロードして
adb install termux.apk

# 起動
adb shell am start -n com.termux/com.termux.app.TermuxActivity
```

### Step 3: Androidの省電力を全部殺す

ここが一番重要。ゲーミングスマホはバックグラウンドプロセスを即殺する設計だから、何もしないとTermuxがすぐ死ぬ。

```bash
# Doze無効化
adb shell dumpsys deviceidle disable

# Termuxをバックグラウンド許可
adb shell dumpsys deviceidle whitelist +com.termux
adb shell cmd appops set com.termux RUN_IN_BACKGROUND allow

# WiFiスリープ防止
adb shell settings put global wifi_sleep_policy 2

# 常時点灯（画面壊れてるから問題なし）
adb shell svc power stayon true
```

画面壊れてるから常時点灯でもバーンインを気にしなくていい。**壊れてることがメリットになる瞬間。**

### Step 4: Node.jsインストール

```bash
# Termux内で
pkg update && pkg upgrade -y
pkg install nodejs -y
node -v  # v22以上必要
```

### Step 5: OpenClawインストール

```bash
npm install -g openclaw
```

#### ハマりポイント: llama-cppのビルド

OpenClawはllama-cppを依存に持ってる。ARMでのC++コンパイルが走るので、ビルドツールが必要：

```bash
pkg install cmake -y
```

10〜20分かかる。ゲーミングスマホの冷却ファンがフル回転する。本来の用途とは違う意味で。

#### ハマりポイント: /tmp問題

Termuxは `/tmp` がRead-onlyファイルシステム。OpenClawが `/tmp/openclaw` を作ろうとしてエラーになる。

```bash
# 全distファイルのパスを書き換え
find /data/data/com.termux/files/usr/lib/node_modules/openclaw/dist/ \
  -type f -name '*.js' \
  -exec sed -i "s|/tmp/openclaw|$PREFIX/tmp/openclaw|g" {} +

mkdir -p $PREFIX/tmp/openclaw
```

力技だが動く。

### Step 6: セットアップと起動

```bash
openclaw onboard
```

`--install-daemon` は使わない。Androidではdaemonインストールが非対応。

```bash
# フォアグラウンドで起動
openclaw gateway run
```

### Step 7: 常駐化

SSHが切れても生き続けるように、tmuxで包む：

```bash
pkg install tmux -y
termux-wake-lock
tmux new -s oc
while true; do openclaw gateway run; sleep 3; done
# Ctrl+B, D でデタッチ
```

クラッシュしても3秒で自動復帰。再接続は `tmux attach -t oc`。

### Step 8: Tailscale（オプション）

外出先からもSSHしたいなら、**Tailscale公式Androidアプリ**を入れる。

⚠️ Termux/proot内のtailscaledは動かない（netlinkrib permission denied）。Androidアプリ版を使うこと。

```bash
adb install tailscale.apk
```

アプリからログインすれば、Tailscaleネットワーク経由でSSHできる。

## ハマりポイントまとめ

| 問題 | 原因 | 解決策 |
|------|------|--------|
| SSH切れる | バックグラウンドプロセス殺し | 設定でバックグラウンド全許可 + wake-lock |
| /tmp作れない | Termuxの制限 | sedで全ファイル書き換え |
| gateway start失敗 | Android非対応 | `gateway run` で直接起動 |
| Tailscaled動かない | proot/SELinuxの制限 | Android公式アプリ版を使う |
| WiFi切れる | スリープ設定 | wifi_sleep_policy 2 |
| llama-cppビルド失敗 | cmake不足 | `pkg install cmake` |

## コスト比較

| | Mac Mini M4 | ジャンクAndroid |
|---|---|---|
| **本体** | 7万円〜 | 0〜3,000円 |
| **消費電力** | 30W | 3W |
| **月間電気代** | ~650円 | ~65円 |
| **性能** | オーバースペック | 十分 |
| **やってること** | API叩くだけ | API叩くだけ |

**同じことをやって、コスト1/20以下。**

## サービス化について

現時点ではtmux + whileループの手動運用。自動起動・死活監視については検討中：

- **Termux:Boot** で端末再起動時に自動起動
- cronで `openclaw gateway status` を定期チェック
- 落ちてたら自動復旧

正式なdaemon化はOpenClaw側のAndroidサポート待ち。

## 結論

画面が壊れたスマホは、ゴミじゃない。**SoC、RAM、WiFi、バッテリーが生きてるなら、それは立派なサーバーだ。**

ゲーミングスマホなら冷却もある。画面壊れてても ADB で全部操作できる。常時点灯してもバーンインを気にしなくていい。

Mac Miniを並べる前に、引き出しの奥のジャンクスマホを見てみてほしい。

---

*この記事は、実際に画面バキバキのRedMagic 8でOpenClawを動かして検証した結果です。隣で2号（えちちゃん/GPT）がWhatsAppで元気に喋ってます。*
