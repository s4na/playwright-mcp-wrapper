# Playwright MCP Wrapper 設計書

## 1. 背景と目的

* **課題**
  * Claude Code からブラウザ操作用 *Playwright MCP* を呼ぶ際、
    同一ポートに複数セッションが流れ込むとブラウザ・コンテキストが混線する。
* **目標**
  1. Claude Code のセッション単位（≒会話単位）で **隔離された Playwright MCP** を自動起動。
  2. セッション終了時に当該 Playwright MCP をクリーンに破棄。
  3. 追加のユーザ操作／ラッパースクリプト無しで実現（`claude mcp add` だけで済む）。

---

## 2. 技術要件

| 要件                | 充足策                                                                                                                                           |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| セッション識別           | Streamable-HTTP で `Mcp-Session-Id` ヘッダーを利用。サーバ側が初回 `initialize` 応答で付与 → 以降のリクエストでクライアント（Claude Code）が自動で送信する仕様 ([modelcontextprotocol.io][1]) |
| Claude Code 接続方法  | `claude mcp add --transport http browser http://127.0.0.1:4000/mcp` のみ。ローカル HTTP→HTTP 透過プロキシ。                                                 |
| Playwright MCP 起動 | `npx playwright-mcp --port <dynamic>` ([github.com][2])                                                                                       |
| ポート衝突回避           | SHA-256(hash(sessionId)) を 2byte 抽出し `20000-39999` にマッピング。                                                                                    |
| 高速復帰              | MacOS / Linux 共通、子プロセスは lazy-spawn & keep-alive 60 s idle。                                                                                    |
| セキュリティ            | 127.0.0.1 bind、`Origin` ヘッダー検証、TLS オフロード不要（ローカル限定）。                                                                                           |

---

## 3. 全体アーキテクチャ

```
┌────────────┐         HTTP (Streamable)
│ Claude Code│──▶  Wrapper MCP  ──▶  Playwright MCP  (port = 20xxx)
└────────────┘         ↑ │                 ↑
                       │ └─ ChildProcess ──┘
                       │
                 Session Registry
```

### 3.1 Wrapper MCP (Session Router)

| 機能           | 詳細                                                                                               |
| ------------ | ------------------------------------------------------------------------------------------------ |
| HTTP エンドポイント | `POST /mcp` (JSON-RPC) / `GET /mcp` (SSE)。                                                       |
| セッション管理      | 初回 `initialize` 受信時に `uuid.v4()` を生成し `Mcp-Session-Id` レスポンスヘッダーで返却。以降は同ヘッダーでルーティング。             |
| プロセス生成       | `spawn('npx', ['playwright-mcp', '--port', port], {env, stdio:'ignore'})`                        |
| ルーティング       | `req.headers['mcp-session-id']` で Playwright プロセスを決定し HTTP Proxy。SSE は `http-proxy` で duplex 転送。 |
| クリーンアップ      | SSE / HTTP keep-alive が切れた時点から 60 s 後に SIGTERM→SIGKILL。                                          |

### 3.2 Session Registry

| Key       | Value                        |
| --------- | ---------------------------- |
| sessionId | `{port, childPid, lastSeen}` |

### 3.3 ポート計算アルゴリズム

```ts
const base = 20000;
const span = 20000;       // 20000-39999
const port = base + (parseInt(sha256(sessionId).slice(0,4), 16) % span);
```

---

## 4. コールフロー詳細

1. **Claude Code → Wrapper**

   ```json
   { "jsonrpc":"2.0", "id":1, "method":"initialize", ... }
   ```
2. **Wrapper**

   * `Mcp-Session-Id` が無ければ生成 → `sid`.
   * `port = calcPort(sid)`
   * `spawn playwright-mcp --port=port`（既存なら skip）
   * `res.setHeader('Mcp-Session-Id', sid)` で `initialize` 応答を返却。
3. **以降のリクエスト**

   * Claude Code が `Mcp-Session-Id: sid` を付与　([modelcontextprotocol.io][1])
   * Wrapper は `sid` をキーに対象 Playwright MCP へ単純リバースプロキシ。
4. **シャットダウン**

   * SSE 断線 or `DELETE /mcp?session=<sid>` → Registry から child を terminate。

---

## 5. 実装サンプル（Node.js/TypeScript）

```ts
import crypto from 'crypto';
import { spawn } from 'child_process';
import fastify from 'fastify';
import httpProxy from '@fastify/http-proxy';

const app = fastify();
const registry = new Map<string, {port:number, proc: any, last:number}>();

function calcPort(sid:string){
  const h = crypto.createHash('sha256').update(sid).digest('hex');
  return 20000 + parseInt(h.slice(0,4),16) % 20000;
}

function ensureWorker(sid:string){
  if(registry.has(sid)) return registry.get(sid)!;
  const port = calcPort(sid);
  const proc = spawn('npx',['playwright-mcp','--port',port],{stdio:'ignore'});
  registry.set(sid,{port,proc,last:Date.now()});
  return {port,proc};
}

app.all('/mcp', async (req, res) => {
  let sid = req.headers['mcp-session-id'] as string | undefined;
  if(!sid){ sid = crypto.randomUUID(); res.header('Mcp-Session-Id', sid); }
  const {port} = ensureWorker(sid);
  const target = `http://127.0.0.1:${port}/mcp`;
  return httpProxy.proxy(req.raw, res.raw, { upstream: target });
});

setInterval(() => {
  const now = Date.now();
  for(const [sid,{proc,last}] of registry){
    if(now-last > 60_000){ proc.kill('SIGTERM'); registry.delete(sid); }
  }
}, 30_000);

app.listen({port:4000});
```

---

## 6. テスト計画

| 番号  | テスト内容                   | 期待結果                                                           |
| --- | ----------------------- | -------------------------------------------------------------- |
| T-1 | 2 つの VSCode ターミナルで別会話開始 | Wrapper が異なる `Mcp-Session-Id` を発行し、別ポートの Playwright が 2 つ起動する。 |
| T-2 | 片方の会話を /exit            | 対応する Playwright が 60 s 後に停止。                                   |
| T-3 | 大量並列 (100 会話)           | ポート衝突なし、メモリ使用が上限内。                                             |

---

## 7. セキュリティ・運用

* Wrapper は `127.0.0.1` 限定バインド。外部公開する場合は Nginx + Mutual-TLS 前段推奨。
* `Origin` ヘッダー必須チェックで DNS rebinding 対策 ([modelcontextprotocol.io][1])
* Playwright は `--browser chromium --headless` 固定、Proxy 経由アクセスを明示可。

---

## 8. 導入手順（最短）

```bash
# 1️⃣ Wrapper MCP をインストール
npm i -g wrapper-mcp   # ← 上記サンプルを npm パッケージ化

# 2️⃣ 起動
wrapper-mcp --port 4000 &

# 3️⃣ Claude Code に登録
claude mcp add --transport http browser http://127.0.0.1:4000/mcp
```

---

### まとめ

* **取得可能なセッション情報**: Streamable-HTTP の `Mcp-Session-Id` を利用すれば Claude Code 側で会話単位に自動で付与されるので取得は容易。
* **実現可否**: 上記のようにラッパー MCP を立てて Playwright MCP を動的起動・プロキシするだけで要件を満たせる。OSS コンポーネントのみで実装でき、運用もシンプル。

不明点や追加要件があれば遠慮なく教えてください！

[1]: https://modelcontextprotocol.io/docs/concepts/transports "Transports - Model Context Protocol"
[2]: https://github.com/faruklmu17/playwright_mcp?utm_source=chatgpt.com "faruklmu17/playwright_mcp - GitHub"