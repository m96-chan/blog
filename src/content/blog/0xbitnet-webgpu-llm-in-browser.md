---
title: 'ブラウザでLLMを動かす：WebGPUで1-bit推論エンジンを作った'
description: 'サーバー不要、ブラウザだけでBitNet b1.58を動かすTypeScriptライブラリ「0xBitNet」を作った。WebGPU + WGSLカーネルによる1-bit LLM推論の仕組みと、開発で得た知見を共有する。'
category: 'Tech'
pubDate: '2026-02-20'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

# LLMを「手元」で動かしたい

LLMを使うとき、ほとんどの場合はAPI経由だ。OpenAI、Anthropic、Google — サーバーに投げて結果をもらう。

でも、こんなことを考えたことはないだろうか：

- APIキーの管理がだるい
- オフラインでも使いたい
- プライバシーが気になる（入力がサーバーに送られる）
- 毎回のAPI課金がじわじわ効いてくる

「ローカルで動かせばよくない？」

llama.cppやollamaで動かす手はある。でもそれはネイティブアプリの話。**ブラウザで、URLを開くだけで、LLMが動いたら**面白くないか？

それを実現するために [**0xBitNet**](https://github.com/m96-chan/0xBitNet) を作った。

# 0xBitNetとは

**ブラウザのWebGPUだけでBitNet b1.58（1-bit LLM）を動かすTypeScriptライブラリ。**

```typescript
import { BitNet } from "0xbitnet";

const model = await BitNet.load(
  "https://huggingface.co/microsoft/bitnet-b1.58-2B-4T-gguf/resolve/main/ggml-model-i2_s.gguf"
);

for await (const token of model.generate("量子コンピュータとは")) {
  process.stdout.write(token);
}
```

これだけ。サーバーなし、WASMなし、ネイティブ拡張なし。ブラウザのGPUを直接叩いて推論する。

npmパッケージとして公開済み：

```bash
npm install 0xbitnet
```

# なぜBitNetなのか

通常のLLM（LLaMA等）は重みがfloat16で、7Bモデルだと14GBのVRAMが要る。ブラウザで動かすのは現実的じゃない。

**BitNet b1.58** は違う。重みが **{-1, 0, +1}** の3値（ternary）しかない。つまり1つの重みに2ビットしか要らない。

| モデル | 重み精度 | 2Bモデルの重みサイズ |
|--------|----------|---------------------|
| 通常のLLM | float16（16bit） | ~4 GB |
| BitNet b1.58 | ternary（2bit） | ~0.5 GB |

Microsoftが公開した [BitNet b1.58 2B-4T](https://huggingface.co/microsoft/BitNet-b1.58-2B-4T) は2Bパラメータで約700MBのGGUFファイル。**ブラウザで十分扱えるサイズ**だ。

# WebGPUで推論エンジンを書くということ

## WGSLカーネルを全部自前で書く

PyTorchもTensorFlowも使えない。WebGPUのシェーディング言語 **WGSL** でゼロからカーネルを書く。

- 行列積（ternary × int8）
- RMSNorm
- RoPE（回転位置エンコーディング）
- Softmax
- ReLU²（BitNet特有の活性化関数）

CUDAと違ってwarp shuffleもshared memory atomicsもない。使えるのはworkgroup shared memoryとバリア同期だけ。制約の中で最適化するのがWebGPUカーネル開発の面白さであり辛さだ。

## I2_Sフォーマットの罠

BitNetのGGUFファイルは **I2_S** という独自フォーマットで重みを格納している。これが曲者で：

- 1バイトに4つのternary値が入っている（2bit × 4）
- **ブロックインターリーブ**されている（128要素が32バイトブロックに分散）
- Microsoft/BitNetフォーク独自のレイアウトで、ドキュメントがほぼない

最初は「シーケンシャルにパックされてるだろう」と思って実装したら、出力がめちゃくちゃだった。Eddie-Wang1120/llama.cppのソースコードを読んで初めてインターリーブの構造を理解した。

```
1ブロック = 32バイト = 128要素
バイト[gp]のビット:
  [7:6] = group0 (offset 0)
  [5:4] = group1 (offset 32)
  [3:2] = group2 (offset 64)
  [1:0] = group3 (offset 96)
```

教訓：**バイナリフォーマットは仕様書じゃなく実装を読め。**

## IndexedDBでモデルをキャッシュ

700MBのモデルを毎回ダウンロードするのは非現実的。ブラウザ環境ではIndexedDBにキャッシュして、2回目以降は即座にロードする。

最初はCache APIを使おうとしたが、大きなBlobで失敗した。IndexedDBの方が大容量データに強い。

Node.jsでは `typeof indexedDB !== "undefined"` ガードで自動スキップされるので、環境依存のコードを書く必要はない。

# v0.2.0: Node.jsでも動く

ブラウザだけじゃない。v0.2.0で **Node.js対応** を追加した。

[`webgpu`](https://www.npmjs.com/package/webgpu) npmパッケージ（GoogleのDawnバインディング）を使えば、Node.jsでもWebGPUが使える：

```typescript
import { create, globals } from "webgpu";
import { BitNet } from "0xbitnet";

// WebGPUグローバルを注入
Object.assign(globalThis, globals);

// Dawnデバイスを作成
const gpu = create([]);
const adapter = await gpu.requestAdapter({ powerPreference: "high-performance" });
const device = await adapter!.requestDevice({ /* max limits */ });

// デバイスを渡してロード
const model = await BitNet.load(url, { device });
```

ポイントは `Object.assign(globalThis, globals)` で `GPUBufferUsage` や `GPUMapMode` などのWebGPU定数をグローバルに設定すること。コアライブラリがこれらを `globalThis` から読むので、これがないと動かない。

CLIの例も用意した：

```bash
cd examples/node-cli
npm install && npm start
```

対話的にチャットでき、tok/sのパフォーマンスも表示される。Denoの `--unstable-webgpu` でも動作確認済み。

# アーキテクチャ

```
0xbitnet/
├── packages/core/          # WGSLカーネル + TypeScript API (npm: 0xbitnet)
│   └── src/
│       ├── gpu/            # WebGPUデバイス初期化、バッファプール
│       ├── model/          # GGUF/Safetensorsパーサー、重みローダー
│       ├── nn/             # Transformerレイヤー、Attention、BitLinear
│       ├── shaders/        # WGSLコンピュートシェーダー
│       ├── tokenizer/      # BPEトークナイザー、チャットテンプレート
│       └── worker/         # Web Workerサポート
├── examples/
│   ├── web-chat/           # チャットアプリデモ
│   ├── tl-dr-widget/       # オフライン要約ウィジェット
│   └── node-cli/           # Node.js CLIデモ
```

推論の流れ：

1. GGUFファイルをfetchでダウンロード（IndexedDBにキャッシュ）
2. メタデータからモデル構成とトークナイザーを抽出
3. テンソルをGPUバッファにアップロード
4. WGSLカーネルでTransformerのforward passを実行
5. ロジットをCPUにreadbackしてサンプリング
6. トークンをデコードしてストリーム出力

# 開発で学んだこと

## バイナリのエンディアン、手で確認しろ

GGUFのマジックナンバーで3時間溶かした。"GGUF"のリトルエンディアンuint32は `0x46554747` であって `0x46475547` ではない。ASCIIコードを1文字ずつ書き出して確認するのが一番確実。

## WebGPU on Linux + NVIDIA + Wayland = 地雷

RTX 5090で0.1 tok/sという異常な遅さに悩まされた。原因はChromeがWayland環境でOpenGL ES（ANGLEの互換モード）にフォールバックしていたこと。

**解決策**: `chrome --ozone-platform=x11` でX11モードにするとVulkan WebGPUが有効になり、パフォーマンスが100倍以上改善された。Chromiumのバグ（[crbug/442791440](https://crbug.com/442791440)）が修正されるまではこのワークアラウンドが必要。

## GPUバッファはリークする

WebGPUの `createBuffer` は毎回GPUプロセスへのIPCが走る。最初の実装では1トークンあたり1000回以上呼んでいた。バッファプールとバインドグループキャッシュで大幅に削減したが、uniformバッファのリークはまだ残っている（[#4](https://github.com/m96-chan/0xBitNet/issues/4)）。

# デモ

- **[Live Chat Demo](https://m96-chan.github.io/0xBitNet/chat/)** — ブラウザで開くだけでLLMチャット
- **[TL;DR Widget](https://m96-chan.github.io/0xBitNet/tldr/)** — オフライン要約ウィジェット
- **[GitHub](https://github.com/m96-chan/0xBitNet)** — ソースコード

```bash
npm install 0xbitnet
```

# まとめ

- **BitNet b1.58** の1-bit（ternary）重みなら、ブラウザで扱えるサイズ
- **WebGPU + WGSL** でゼロからカーネルを書けば、ネイティブ並の推論が可能
- **Node.js** でもDawnバインディング経由で同じAPIが使える
- バイナリフォーマットは実装を読め、WebGPUのLinux環境には罠がある

LLMをAPIに頼らず、ユーザーの手元で動かす。それが0xBitNetの目指すところだ。

---

リポジトリ: [github.com/m96-chan/0xBitNet](https://github.com/m96-chan/0xBitNet)
npm: [npmjs.com/package/0xbitnet](https://www.npmjs.com/package/0xbitnet)
