---
title: '海外を視野にいれた家庭内VPNを整備する'
description: 'Tailscale + ASUS Merlin + クラウドインスタンスで、世界中どこからでも自宅ネットワークにアクセスできる環境を構築する'
category: 'Tech'
pubDate: '2026-02-06'
heroImage: '../../assets/vpn-hero.jpg'
---

# はじめに：エンジニアにNordVPNはいらない

海外移住やノマド生活を考えると、まず浮かぶのがVPNだ。

NordVPN、ExpressVPN、Surfshark...月額1,000円前後で「安全にインターネット」ができるらしい。  
でも、エンジニアなら考えてほしい。

**自分で建てた方が安い、速い、自由度が高い。**

商用VPNは便利だが、結局は「他人のサーバー」を経由しているに過ぎない。  
ログを取らないと謳っていても、信じるかどうかはあなた次第だ。

この記事では、**Tailscale + ASUS Merlinルーター + クラウドインスタンス**を組み合わせて、世界中どこからでも自宅ネットワークにアクセスできるVPN環境を構築する方法を紹介する。

# 全体構成

```
┌─────────────────────────────────────────────┐
│                 Tailscale Mesh               │
│                                              │
│  🏠 自宅 (東京)                              │
│  ├── ASUS Router (Merlin) ← Subnet Router   │
│  │   └── 192.168.50.0/24 を広告             │
│  ├── PC / NAS / IoTデバイス                  │
│  │                                           │
│  ☁️ クラウド                                  │
│  ├── OCI 大阪 ← 日本Exit Node               │
│  ├── AWS Frankfurt ← EU Exit Node           │
│  └── (追加し放題)                             │
│                                              │
│  📱 外出先                                    │
│  ├── ノートPC                                │
│  └── スマホ                                  │
└─────────────────────────────────────────────┘
```

**ポイント：**
- 自宅のルーターがSubnet Routerになり、外出先から自宅LANに直接アクセス可能
- 各国のクラウドインスタンスをExit Nodeにすれば、その国のIPアドレスでブラウジング可能
- 全デバイスがメッシュ接続され、P2Pで直接通信（中継サーバーを経由しない）

# Tailscaleとは

## WireGuardの面倒を全部やってくれるやつ

[Tailscale](https://tailscale.com/)は、**WireGuardベースのメッシュVPN**だ。

WireGuardそのものは素晴らしいプロトコルだが、鍵の配布、NATトラバーサル、ルーティングの設定など、自分でやると結構面倒くさい。  
Tailscaleは、その面倒な部分を全部やってくれる。

## 主な特徴

| 特徴 | 説明 |
|------|------|
| **メッシュ接続** | 全デバイスがP2Pで直接通信。中央サーバーを経由しない |
| **NAT越え** | ほぼどんなネットワーク環境でも接続できる |
| **MagicDNS** | デバイス名で名前解決できる（`my-pc.tail1234.ts.net`） |
| **ACL** | 細かいアクセス制御が可能 |
| **無料枠** | 個人利用なら100デバイスまで無料 |

## インストール

ほぼ全プラットフォームに対応している。

```bash
# Linux
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# macOS / Windows / iOS / Android
# → 公式サイトからアプリをインストール
```

これだけで、そのデバイスはTailscaleネットワークに参加する。  
面倒な鍵交換も、ポート開放も一切不要。

# ASUS Merlinルーターの設定

## なぜルーターにTailscaleを入れるのか

自宅のPCにTailscaleを入れるだけでは、そのPCにしかアクセスできない。  
しかし、**ルーターにTailscaleを入れてSubnet Routerにする**と、自宅LAN全体（`192.168.50.0/24`）に外からアクセスできるようになる。

NAS、プリンター、IoTデバイス、監視カメラ...  
全部、外出先から自宅にいるのと同じようにアクセス可能だ。

## ASUS Merlinとは

[Asuswrt-Merlin](https://www.asuswrt-merlin.net/)は、ASUSルーターの**カスタムファームウェア**だ。  
純正ファームウェアをベースに、以下のような拡張がされている：

- **Entware対応**: Linuxパッケージマネージャーが使える
- **カスタムスクリプト**: 起動時やイベント時にシェルスクリプトを実行可能
- **SSH**: 当然使える

つまり、**ルーターをLinuxサーバーのように扱える**。

## セットアップ手順

### 1. USBメモリの準備

EntwareはUSBストレージにインストールする。  
**ext4でフォーマット**したUSBメモリをルーターに挿す。

```bash
# ルーターのWebUIから、USB → フォーマット でext4を選択
# または SSH で:
umount /tmp/mnt/sda1
mkfs.ext4 /dev/sda1
```

### 2. Entwareのインストール

SSHでルーターに接続して：

```bash
amtm
```

`amtm`はMerlinのパッケージマネージャーだ。メニューからEntwareをインストールする。

### 3. Tailscaleのインストール

```bash
# Entware経由でインストール
opkg update
opkg install tailscale
```

### 4. Tailscaleの起動とSubnet Router設定

```bash
# 初回起動
tailscaled --state=/opt/var/tailscale/tailscaled.state \
           --socket=/var/run/tailscale/tailscaled.sock &

# Subnet Routerとして起動
tailscale up --advertise-routes=192.168.50.0/24
```

初回は認証URLが表示されるので、ブラウザで開いてTailscaleの管理画面から承認する。

**重要**: Tailscaleの管理画面（admin console）で、このデバイスのSubnet Routeを**承認**する必要がある。

### 5. 自動起動の設定

ルーターが再起動しても自動でTailscaleが立ち上がるように設定する。

```bash
# /jffs/scripts/services-start に追加
cat << 'EOF' >> /jffs/scripts/services-start
# Tailscale auto-start
sleep 30
/opt/sbin/tailscaled --state=/opt/var/tailscale/tailscaled.state \
                     --socket=/var/run/tailscale/tailscaled.sock &
sleep 5
/opt/bin/tailscale up --advertise-routes=192.168.50.0/24
EOF

chmod +x /jffs/scripts/services-start
```

`sleep 30`は、ネットワークが完全に立ち上がるのを待つためだ。  
これがないと、Tailscaleが起動しても接続に失敗する場合がある。

# 世界中にExit Nodeを建てる

## Exit Nodeとは

Tailscaleの**Exit Node**機能を使うと、そのノードを経由してインターネットにアクセスできる。  
つまり、**その国のIPアドレスでブラウジング**できる。

用途は明確だ：

- 🇯🇵 **日本のExit Node** → 海外から日本のサービスにアクセス（Abema、TVer、etc.）
- 🇪🇺 **EUのExit Node** → 日本からEU限定サービスにアクセス
- 🇺🇸 **USのExit Node** → 米国限定コンテンツにアクセス

## 無料クラウドインスタンスを活用する

### Oracle Cloud Infrastructure（OCI）

**Always Free枠**で、以下が永久無料：

- **AMD Micro**: 1 OCPU, 1GB RAM × 2台
- **Arm A1 Flex**: 最大4 OCPU, 24GB RAM（空きがあれば）

リージョンは**東京・大阪**が選択可能。  
日本IP用のExit Nodeに最適だ。

```bash
# OCIインスタンスでTailscaleをセットアップ
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-exit-node
```

管理画面でExit Nodeを承認すれば完了。

**注意**: OCIのセキュリティリスト（ファイアウォール）で、**UDPポート41641**を開けておくとP2P接続が安定する。開けなくてもリレー経由で動くが、速度が落ちる。

### AWS Free Tier

**t2.micro**が12ヶ月無料。リージョンが豊富なので、EU（Frankfurt）やUS（Virginia）に建てるのに適している。

```bash
# 同様にセットアップ
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-exit-node
```

**Elastic IPを割り当てる**のを忘れずに。インスタンスを停止→起動するとIPが変わってしまう。

### 月額ランニングコスト

| ノード | 場所 | コスト |
|--------|------|--------|
| OCI Micro ×2 | 日本（大阪） | **無料** |
| OCI A1 Flex | 日本 | **無料**（空きがあれば） |
| AWS t2.micro | EU（Frankfurt） | **無料**（12ヶ月） |
| ConoHa 512MB | 日本 | **月300円** |

そう、ほぼ**タダ**である。

NordVPNに月1,000円払うのがバカらしくなるだろう？

# 使い方

## Exit Nodeの切り替え

外出先や海外から、使いたいExit Nodeを切り替えるだけ。

```bash
# 日本IP経由でブラウジング
tailscale set --exit-node=oci-osaka

# EU IP経由でブラウジング
tailscale set --exit-node=aws-frankfurt

# Exit Nodeを解除（直接接続に戻す）
tailscale set --exit-node=
```

スマホアプリからも、タップ一つで切り替えられる。

## 自宅LANへのアクセス

Subnet Routerが動いていれば、自宅のローカルIPにそのままアクセスできる。

```bash
# 自宅のNASにSSH
ssh admin@192.168.50.100

# 自宅のルーター管理画面
open https://192.168.50.1:8443
```

海外のカフェからでも、自宅にいるのと全く同じ操作ができる。

# 商用VPN vs 自前VPN

| 項目 | NordVPN等 | Tailscale + クラウド |
|------|-----------|---------------------|
| **月額** | ~1,000円 | ほぼ無料 |
| **速度** | 共有サーバーなので混雑時遅い | 自分専用なので安定 |
| **ログ** | 「取ってない」と信じるしかない | 自分のサーバーなので確実 |
| **IP選択** | 決まった国から選ぶ | 好きな国にインスタンスを建てるだけ |
| **自宅LAN** | アクセス不可 | Subnet Routerで完全アクセス |
| **カスタマイズ** | ほぼ不可 | 何でもできる |
| **難易度** | ボタンひとつ | Linuxの基本知識が必要 |

唯一の欠点は、**セットアップにLinuxの知識が必要**なことだ。  
だが、この記事を読んでいるあなたなら問題ないだろう。

# まとめ

- **Tailscale**はWireGuardベースのメッシュVPN。設定が簡単で、無料枠が太い
- **ASUS Merlin**ルーターをSubnet Routerにすれば、自宅LAN全体に外からアクセス可能
- **OCI / AWS の無料枠**を活用すれば、世界各国にExit Nodeを建てられる
- ランニングコストは**ほぼゼロ**

エンジニアなら、商用VPNに金を払う必要はない。  
自分のインフラは自分で建てる。それがエンジニアだ。

**NordVPNはいらない。**
