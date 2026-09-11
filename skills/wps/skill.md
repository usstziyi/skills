---
name: kdocs-summary
description: 金山文档内容总结 Skill。通过 Agent API Key 认证，调用 V7 内容抽取接口读取文档并生成 LLM 结构化摘要。当用户提到"总结文档"、"文档摘要"、"kdocs 总结"、"金山文档总结"时使用。
---

# 金山文档内容总结 Skill

通过环境变量 `AGENT_API_KEY` 鉴权，调用 WPS V7 接口读取金山文档，再由 LLM 输出结构化摘要。

## Agent 执行指南

用户要求总结金山文档时，按下列步骤执行（开箱即用入口）：

1. **确认文档 URL**  
   需提供 `365.kdocs.cn` / `kdocs.cn` 的 `/l/...` 链接或纯 `link_id`；缺失则停止并请用户补充。

2. **确认依赖**  
   `python -c "import requests; print('requests ok')"`  
   失败则 `pip install requests` 后重试。

3. **确认 `AGENT_API_KEY`**（只校验存在与长度，**禁止**回显完整值）  
   - 校验（Windows PowerShell）：
     ```powershell
     $k = $env:AGENT_API_KEY
     if (-not $k) { $k = [Environment]::GetEnvironmentVariable("AGENT_API_KEY", "User") }
     if ($k) { "AGENT_API_KEY is set (len=$($k.Length))" } else { "AGENT_API_KEY is NOT set" }
     ```
   - 校验（macOS/Linux）：
     ```bash
     if [ -n "$AGENT_API_KEY" ]; then echo "AGENT_API_KEY is set (len=${#AGENT_API_KEY})"; else echo "AGENT_API_KEY is NOT set"; fi
     ```
   - **已有 User 级、会话为空（Windows）**：先加载再执行  
     `$env:AGENT_API_KEY = [Environment]::GetEnvironmentVariable("AGENT_API_KEY", "User")`
   - **未设置**：立即停止。**仅**提示用户自行将密钥配置到环境变量后重试；**禁止**引导用户在对话输入框粘贴/提供密钥。  
     提示内容须包含密钥格式 `apik:<sk_id>.<secret_key>`，以及下列配置路径与方法（按用户 OS 择要给出）：
     - **Windows（推荐，用户级持久化）**：PowerShell 执行  
       `[Environment]::SetEnvironmentVariable("AGENT_API_KEY", "你的密钥", "User")`  
       完成后**重启 Cursor**（或新开终端）；也可：系统设置 → 系统 → 关于 → 高级系统设置 → 环境变量 → 用户变量 → 新建 `AGENT_API_KEY`
     - **macOS（zsh）**：编辑 `~/.zshrc`，末尾追加 `export AGENT_API_KEY="你的密钥"`，执行 `source ~/.zshrc`，并重启 Cursor
     - **Linux（bash）**：编辑 `~/.bashrc`，末尾追加同上，`source ~/.bashrc`，并重启 Cursor  
     配置后勿在对话中回显密钥；禁止 `--api-key` / 写入 Skill 或仓库。

4. **写入临时脚本**  
   将下方「内置 Python 脚本」写入系统临时目录（如 `$env:TEMP\_tmp_kdocs_summary.py`），勿写入仓库。

5. **执行**（stdout=文档正文，stderr=诊断日志；以 stdout 做总结）  
   ```bash
   python "<临时脚本路径>" --url "<文档URL>" --format md
   ```
   | 参数 | 必填 | 说明 |
   |------|:----:|------|
   | `--url` | ✅ | 分享链接（可带查询参数/锚点） |
   | `--format` | ❌ | **输出**格式：`md`（默认）/ `txt` / `json` |
   | `--content-format` | ❌ | **API 抽取**格式：`markdown`（默认）/ `plain` |
   | `--output` | ❌ | 写入文件；默认 stdout |

6. **清理**  
   等价 `try { 执行 } finally { 删除临时脚本 }`，失败也必须删除。

7. **LLM 总结并输出**（默认 Markdown；用户另有指定则从其指定）  
   若 stderr 出现 `[WARN] content via https://openapi.wps.cn` 且随后有 `content fallback to https://api.wps.cn`，视为**成功**，勿中断。  
   若内容长度为 0，提示检查文档权限或格式。

### 摘要输出模板

```markdown
# 文档总结：{文件名}

## 基本信息
- 文件名：{file_name}
- 类型：{源格式}
- 创建者：{creator}

## 核心要点
1. {要点1}
2. {要点2}
3. ...

## 关键信息
{从文档中提取的关键数据、结论或决策}

## 一句话总结
{用一句话概括文档核心内容}
```

## 内置 Python 脚本

> 写入系统临时目录执行；无论成败，`finally` 中删除。

```python
#!/usr/bin/env python3
"""kdocs_summary.py - 通过 Agent API Key + V7 内容抽取接口读取金山文档内容"""

import argparse
import io
import json
import os
import re
import sys
import time

if sys.platform == "win32":
    sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding="utf-8")
    sys.stderr = io.TextIOWrapper(sys.stderr.buffer, encoding="utf-8")

import requests


class KDocsSummaryReader:
    """通过 Agent API Key + V7 内容抽取接口读取金山文档内容"""

    TOKEN_URL = "https://account.wps.cn/api/authorization/agent/v1/token"
    V7_BASE = "https://openapi.wps.cn"

    def __init__(self):
        self.api_key = os.environ.get("AGENT_API_KEY", "").strip()
        if not self.api_key:
            raise ValueError(
                "未检测到环境变量 AGENT_API_KEY。"
                "请自行配置到系统/用户环境变量后重试（勿在对话中粘贴密钥）。"
            )
        self._token = None
        self._expires_at = 0

    @property
    def headers(self) -> dict:
        """返回带 AccessToken 的请求头，自动刷新过期 Token"""
        if not self._token or time.time() >= self._expires_at:
            self._refresh_token()
        return {"Authorization": f"Bearer {self._token}"}

    def _refresh_token(self):
        """用 API Key 换取 AccessToken（JWT, 12h有效）"""
        resp = requests.post(self.TOKEN_URL, json={
            "grant_type": "api_key",
            "api_key": self.api_key,
        }, timeout=30)
        resp.raise_for_status()
        result = resp.json()
        if result.get("result") != "ok":
            raise RuntimeError(f"换取 Token 失败: {result.get('msg', result)}")
        data = result["data"]
        self._token = data["access_token"]
        self._expires_at = time.time() + data["expires_in"] - 1800

    @staticmethod
    def extract_link_id(url: str) -> str:
        """从金山文档 URL 中提取 link_id，支持带查询参数和锚点的 URL"""
        url = url.split("?")[0].split("#")[0].rstrip("/")
        match = re.search(r'/l/([a-zA-Z0-9]+)', url)
        if match:
            return match.group(1)
        if re.match(r'^[a-zA-Z0-9]+$', url.strip()):
            return url.strip()
        raise ValueError(f"无法从 URL 提取 link_id: {url}")

    def get_link_meta(self, link_id: str) -> dict:
        """通过 link_id 获取 drive_id 和 file_id"""
        resp = requests.get(
            f"{self.V7_BASE}/v7/links/{link_id}/meta",
            headers=self.headers,
            timeout=30,
        )
        resp.raise_for_status()
        data = resp.json()
        if data.get("code") != 0:
            raise RuntimeError(f"获取链接信息失败: {data.get('msg', data)}")
        return data["data"]

    def get_file_content(self, drive_id: str, file_id: str, content_format: str = "markdown") -> dict:
        """调用 V7 内容抽取接口读取文档正文（本 Skill 仅使用 format: markdown | plain）"""
        path = f"/v7/drives/{drive_id}/files/{file_id}/content"
        params = {"format": content_format}
        bases = [self.V7_BASE]
        # openapi 网关若未放通 content，回退 api.wps.cn
        if "openapi.wps.cn" in self.V7_BASE:
            bases.append("https://api.wps.cn")
        last_err = None
        for base in bases:
            resp = requests.get(f"{base}{path}", headers=self.headers, params=params, timeout=60)
            if resp.status_code >= 400:
                last_err = f"{resp.status_code} {resp.text[:200]}"
                print(f"[WARN] content via {base} failed: {last_err}", file=sys.stderr)
                continue
            data = resp.json()
            if data.get("code") != 0:
                last_err = data.get("msg", data)
                continue
            if base != self.V7_BASE:
                print(f"[INFO] content fallback to {base}", file=sys.stderr)
            return data["data"]
        raise RuntimeError(f"内容抽取失败: {last_err}")

    def read_document(self, url: str, content_format: str = "markdown") -> dict:
        """完整流程：URL → link_id → drive_id/file_id → 内容抽取"""
        link_id = self.extract_link_id(url)
        print(f"[INFO] link_id: {link_id}", file=sys.stderr)

        link_meta = self.get_link_meta(link_id)
        drive_id = link_meta["drive_id"]
        file_id = link_meta["file_id"]
        creator = link_meta.get("creator", {}).get("name", "")
        print(f"[INFO] drive_id={drive_id}, file_id={file_id}", file=sys.stderr)

        content_data = self.get_file_content(drive_id, file_id, content_format=content_format)
        file_name = content_data.get("file_info", {}).get("name", "")
        if not file_name:
            first_line = (content_data.get(content_format) or "").split("\n", 1)[0]
            file_name = re.sub(r'^[#\s]+', '', first_line).strip()[:60] or f"doc_{link_id}"
        content = content_data.get(content_format) or content_data.get("markdown") or content_data.get("plain") or ""
        print(f"[INFO] 文件: {file_name}, 内容长度: {len(content)} chars", file=sys.stderr)

        return {
            "file_name": file_name,
            "creator": creator,
            "drive_id": drive_id,
            "file_id": file_id,
            "link_id": link_id,
            "src_format": content_data.get("src_format", ""),
            "content": content,
        }


def format_output(result: dict, fmt: str) -> str:
    """根据指定格式输出内容"""
    if fmt == "json":
        return json.dumps(result, ensure_ascii=False, indent=2)
    elif fmt == "txt":
        header = f"文件名: {result['file_name']}\n创建者: {result['creator']}\n{'='*60}\n\n"
        return header + result["content"]
    else:
        header = (
            f"---\n"
            f"文件名: {result['file_name']}\n"
            f"创建者: {result['creator']}\n"
            f"源格式: {result['src_format']}\n"
            f"---\n\n"
        )
        return header + result["content"]


def main():
    parser = argparse.ArgumentParser(description="金山文档内容读取（V7 内容抽取接口）")
    parser.add_argument("--url", required=True, help="金山文档分享链接")
    parser.add_argument("--format", choices=["md", "txt", "json"], default="md",
                        help="输出格式（默认: md）")
    parser.add_argument("--content-format", choices=["markdown", "plain"], default="markdown",
                        help="API 内容抽取格式（默认: markdown；本 Skill 仅支持 markdown/plain）")
    parser.add_argument("--output", help="输出文件路径（不指定则输出到 stdout）")
    args = parser.parse_args()

    try:
        reader = KDocsSummaryReader()
        result = reader.read_document(args.url, content_format=args.content_format)
        output = format_output(result, args.format)

        if args.output:
            from pathlib import Path
            out_path = Path(args.output)
            out_path.parent.mkdir(parents=True, exist_ok=True)
            out_path.write_text(output, encoding="utf-8")
            print(f"[INFO] 已保存到: {out_path.absolute()}", file=sys.stderr)
        else:
            print(output)

    except Exception as e:
        print(f"[ERROR] {e}", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

## 错误处理

| 错误 | 处理 |
|------|------|
| 缺少 URL | 请用户补充分享链接 |
| `ModuleNotFoundError: requests` | `pip install requests` |
| 未检测到 `AGENT_API_KEY` | 停止；按执行指南第 3 步提示自行配置环境变量（勿索要对话内密钥） |
| `InvalidApiKey` | 检查/重置密钥后更新环境变量 |
| 获取链接信息失败 | 查鉴权与文档权限 |
| openapi content `400000004` + 已 fallback | **正常**，继续总结 |
| 内容抽取最终失败 / 正文长度为 0 | 查权限与格式 |
| `notCompanyMember` | 开放文档权限或 Agent 加入企业 |