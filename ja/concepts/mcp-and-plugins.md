---
title: MCP 統合とプラグインシステム
created: 2026-04-07
updated: 2026-04-07
type: concept
tags: [architecture, mcp, plugins, extensibility]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/mcp-and-plugins.md
---

# MCP 統合とプラグインシステム

## 設計原理

Hermes は **MCP（Model Context Protocol）**と**プラグインシステム**で拡張性を実現します。外部ツール接続とカスタム動作が可能です。

## MCP 統合

```python
# tools/mcp_tool.py (~2176 行)

class MCPServerTask:
    """MCP サーバータスク"""
    
    def __init__(self, config: dict):
        self.servers = {}
        self.tools = {}
    
    async def connect_server(self, name: str, config: dict):
        """MCP サーバーに接続"""
        transport = config.get("transport", "stdio")
        
        if transport == "stdio":
            process = await asyncio.create_subprocess_exec(
                *config["command"],
                stdin=asyncio.subprocess.PIPE,
                stdout=asyncio.subprocess.PIPE,
            )
            self.servers[name] = {
                "process": process,
                "transport": transport,
            }
        elif transport == "http":
            self.servers[name] = {
                "url": config["url"],
                "transport": transport,
            }
        
        # サーバーツールを取得
        tools = await self._list_tools(name)
        for tool in tools:
            self.tools[f"{name}:{tool['name']}"] = tool
    
    async def call_tool(self, tool_name: str, args: dict) -> dict:
        """MCP ツールを呼び出す"""
        server_name, tool_name = tool_name.split(":", 1)
        server = self.servers[server_name]
        
        if server["transport"] == "stdio":
            return await self._call_stdio_tool(server, tool_name, args)
        elif server["transport"] == "http":
            return await self._call_http_tool(server, tool_name, args)
```

### MCP OAuth サポート

```python
# tools/mcp_oauth.py

async def authenticate_mcp_server(server_config: dict) -> dict:
    """MCP サーバー OAuth 認証"""
    auth_type = server_config.get("auth", {}).get("type")
    
    if auth_type == "oauth":
        # OAuth フローを実装
        auth_url = server_config["auth"]["url"]
        client_id = server_config["auth"]["client_id"]
        # ...
        return {"access_token": token, "expires_at": expires}
    
    elif auth_type == "api_key":
        return {"api_key": server_config["auth"]["api_key"]}
    
    return {}
```

## プラグインシステム

```python
# hermes_cli/plugins.py

class Plugin:
    """プラグイン基底クラス"""
    
    name: str = ""
    version: str = "1.0.0"
    description: str = ""
    
    def on_load(self):
        """プラグインロード時呼び出し"""
        pass
    
    def on_unload(self):
        """プラグインアンロード時呼び出し"""
        pass

# フックシステム
_HOOKS = {
    "on_session_start": [],
    "pre_llm_call": [],
    "post_llm_call": [],
    "on_tool_call": [],
    "on_session_end": [],
}

def register_hook(hook_name: str, callback: callable):
    """フックコールバックを登録"""
    if hook_name in _HOOKS:
        _HOOKS[hook_name].append(callback)

def invoke_hook(hook_name: str, **kwargs) -> list:
    """フックを呼び出す"""
    results = []
    for callback in _HOOKS.get(hook_name, []):
        try:
            result = callback(**kwargs)
            results.append(result)
        except Exception as e:
            logger.warning(f"Hook {hook_name} failed: {e}")
    return results
```

### メモリプラグイン

```python
# plugins/memory/__init__.py

class MemoryPlugin(Plugin):
    """メモリプラグイン（Honcho 統合）"""
    
    name = "honcho-memory"
    
    def on_session_start(self, session_id: str, **kwargs):
        """セッション開始時にキャッシュをウォームアップ"""
        self._warm_cache(session_id)
    
    def pre_llm_call(self, user_message: str, **kwargs):
        """LLM 呼び出し前にコンテキスト注入"""
        context = self._fetch_context(user_message)
        return {"context": context}
    
    def on_session_end(self, messages: list, **kwargs):
        """セッション終了時に永続化"""
        self._persist_session(messages)
```

## プラグイン CLI

```bash
# プラグイン管理
hermes plugins list           # インストール済みプラグイン一覧
hermes plugins install <name> # プラグインインストール
hermes plugins remove <name>  # プラグイン削除
hermes plugins update <name>  # プラグイン更新
```

## 設定

```yaml
# ~/.hermes/config.yaml
mcp_servers:
  filesystem:
      command: ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/root/work"]
    github:
      command: ["npx", "-y", "@modelcontextprotocol/server-github"]
      env:
        GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"

plugins:
  enabled:
    - honcho-memory
    - custom-plugin
```

## 優位性分析

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | Cursor | Claude Code |
|------|--------|--------|-------------|
| MCP サポート | ✅ 完全 | ✅ | ✅ |
| MCP OAuth | ✅ | ❌ | ✅ |
| プラグインシステム | ✅ フックシステム | ❌ | ❌ |
| カスタムツール | ✅ 登録テーブル | ❌ | ❌ |
| プラグイン CLI | ✅ | N/A | N/A |

## 関連ページ

- [[tool-registry-architecture]] — プラグインは registry.register() でツール登録
- [[hook-system-architecture]] — プラグインフックシステムとゲートウェイイベントフックは相補
- [[model-tools-dispatch]] — MCP ツールは discover 機構で編成層に統合

## 関連ファイル

- `tools/mcp_tool.py` — MCP サーバータスク
- `tools/mcp_oauth.py` — MCP OAuth
- `hermes_cli/plugins.py` — プラグインシステム
- `plugins/` — プラグインディレクトリ
