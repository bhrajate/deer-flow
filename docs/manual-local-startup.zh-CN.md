# 不通过 Make 手动启动 DeerFlow 本地组件

本文说明如何在本机分别启动 DeerFlow 的 Gateway、Frontend 和 Nginx，适合需要独立调试各组件的开发场景。

> 默认使用三个终端，启动顺序为 **Gateway → Frontend → Nginx**。每个新终端都是独立的 Shell，不会继承其他终端中执行的 `export` 或 `source`。

## 前置条件

请先准备：

- Python 3.12 与 [uv](https://docs.astral.sh/uv/)
- Node.js 22+ 与 pnpm
- Nginx
- 仓库根目录下有效的 `config.yaml`
- 模型配置所引用的环境变量，例如 `OPENAI_API_KEY`

本文假设仓库路径为：

```text
/home/glory/code/github/deer-flow
```

如果实际路径不同，请修改下文的 `DEER_FLOW_PROJECT_ROOT`。

## 配置 `config.yaml` 与 `.env`

DeerFlow 推荐将非敏感应用配置写入仓库根目录的 `config.yaml`，将 API Key、Token、数据库连接串等敏感值写入根目录的 `.env`。这两个本地文件都已被仓库根目录的 `.gitignore` 忽略，不应提交到 Git。

### 1. 创建本地配置文件

在仓库根目录执行：

```bash
cd /home/glory/code/github/deer-flow

# -n 可避免覆盖已有配置。
cp -n config.example.yaml config.yaml

# 创建密钥文件并限制为仅当前用户可读写。
touch .env
chmod 600 .env
```

如果 `config.yaml` 已存在，请直接编辑现有文件，不要再次从示例覆盖。完整配置项可参考 [`config.example.yaml`](../config.example.yaml) 和[后端配置说明](../backend/docs/CONFIGURATION.md)。

### 2. 在 `.env` 中保存密钥

例如使用 OpenAI 时，在根目录 `.env` 中加入：

```dotenv
OPENAI_API_KEY=替换为实际密钥
```

使用其他模型供应商时，可定义对应变量，例如：

```dotenv
ANTHROPIC_API_KEY=替换为实际密钥
DEEPSEEK_API_KEY=替换为实际密钥
```

注意：

- 不要把真实密钥直接写进本文、终端历史、日志或提交到 Git。
- `.env` 使用 `KEY=value` 格式，等号两侧不要留空格。
- 因为本文使用 Shell 的 `source .env` 加载文件，值中若含空格或 Shell 特殊字符，应使用单引号，例如 `TOKEN='实际值'`。
- 不要在值前后添加不属于密钥本身的引号或空格。

### 3. 在 `config.yaml` 中引用环境变量

`config.yaml` 的任意字段值都可以使用完整的 `$变量名` 引用环境变量。以下是一个使用 OpenAI Chat Completions 的最小模型示例：

```yaml
models:
  - name: gpt-4.1
    display_name: GPT-4.1
    use: langchain_openai:ChatOpenAI
    model: gpt-4.1
    api_key: $OPENAI_API_KEY
    request_timeout: 600.0
    max_retries: 2
    supports_vision: true
```

使用兼容 OpenAI API 的代理时，可增加 `base_url`：

```yaml
models:
  - name: my-model
    display_name: My Model
    use: langchain_openai:ChatOpenAI
    model: 代理实际支持的模型或部署名称
    base_url: https://代理地址/v1
    api_key: $OPENAI_API_KEY
    request_timeout: 600.0
    max_retries: 2
    use_responses_api: false
```

这里有几个重要规则：

- 使用 `$OPENAI_API_KEY`，不要使用 `${OPENAI_API_KEY}`。当前配置解析器把以 `$` 开头的整个剩余字符串当作变量名。
- 环境变量引用必须占据完整字段值；不支持在 `https://$HOST/v1` 这样的字符串中插值。
- 引用的变量不存在时，Gateway 会在加载配置时报告 `Environment variable ... not found` 并停止启动。
- `model` 必须是供应商或代理实际支持的模型名；Azure/OpenAI 代理有时要求填写部署名称。
- 只有上游明确支持 OpenAI Responses API 时才设置 `use_responses_api: true` 和 `output_version: responses/v1`。多数只兼容 `/v1/chat/completions` 的代理应将其设为 `false`，并删除 `output_version`。
- `models` 至少需要一个有效条目；未显式选择模型时，列表中的首个模型通常作为默认模型。

### 4. 配置文件路径与加载优先级

本文在 Gateway 启动命令中显式设置：

```bash
export DEER_FLOW_PROJECT_ROOT=/home/glory/code/github/deer-flow
export DEER_FLOW_CONFIG_PATH="$DEER_FLOW_PROJECT_ROOT/config.yaml"
export DEER_FLOW_HOME="$DEER_FLOW_PROJECT_ROOT/backend/.deer-flow"
```

其中：

- `DEER_FLOW_PROJECT_ROOT`：项目根目录，也是相对运行路径的解析基准。
- `DEER_FLOW_CONFIG_PATH`：明确指定要加载的 `config.yaml`。
- `DEER_FLOW_HOME`：运行期状态目录。

Gateway 查找配置文件的优先级为：代码显式参数、`DEER_FLOW_CONFIG_PATH`、项目根目录或当前目录下的 `config.yaml`、兼容旧目录结构的兜底位置。手动启动时显式设置前两个路径变量最可靠。

根目录 `.env` 中的变量通过本文启动命令中的以下语句导出：

```bash
set -a
source "$DEER_FLOW_PROJECT_ROOT/.env"
set +a
```

如果当前 Shell 已经 `export` 了同名变量，该值会在应用调用 `load_dotenv(override=False)` 时保持不变。因此建议在启动前检查是否残留了旧变量：

```bash
env | grep -E '^(DEER_FLOW_|OPENAI_API_KEY=|ANTHROPIC_API_KEY=|DEEPSEEK_API_KEY=)' \
  | sed 's/=.*$/=<已设置>/'
```

该命令只显示变量名及是否已设置，不显示实际值。

### 5. 启动前验证配置

先加载 `.env`，然后让应用解析完整配置：

```bash
cd /home/glory/code/github/deer-flow
set -a
source .env
set +a

cd backend
DEER_FLOW_PROJECT_ROOT=.. \
DEER_FLOW_CONFIG_PATH=../config.yaml \
PYTHONPATH=. \
uv run python -c \
  "from deerflow.config.app_config import get_app_config; c=get_app_config(); print('config loaded, models:', len(c.models))"
```

成功时会输出模型数量，不会打印密钥。如果某个 `$变量名` 未定义，命令会直接指出缺失的变量名。

也可单独确认某个变量是否存在而不输出其值：

```bash
if [ -n "${OPENAI_API_KEY:-}" ]; then
  echo 'OPENAI_API_KEY is set'
else
  echo 'OPENAI_API_KEY is missing'
fi
```

## 一次性安装依赖

安装后端依赖：

```bash
cd /home/glory/code/github/deer-flow/backend
uv sync --all-packages
```

安装前端依赖：

```bash
cd /home/glory/code/github/deer-flow/frontend
pnpm install
```

## 1. 启动 Gateway

打开第一个终端：

```bash
export DEER_FLOW_PROJECT_ROOT=/home/glory/code/github/deer-flow
export DEER_FLOW_CONFIG_PATH="$DEER_FLOW_PROJECT_ROOT/config.yaml"
export DEER_FLOW_HOME="$DEER_FLOW_PROJECT_ROOT/backend/.deer-flow"

mkdir -p "$DEER_FLOW_HOME" "$DEER_FLOW_PROJECT_ROOT/backend/sandbox"

# 将根目录 .env 中的变量导出给 Gateway；已由 Shell 导出的同名变量优先。
set -a
source "$DEER_FLOW_PROJECT_ROOT/.env"
set +a

cd "$DEER_FLOW_PROJECT_ROOT/backend"
PYTHONPATH=. \
PYTHONIOENCODING=utf-8 \
PYTHONUTF8=1 \
uv run uvicorn app.gateway.app:app \
  --host 0.0.0.0 \
  --port 8001 \
  --reload
```

显式设置 `DEER_FLOW_PROJECT_ROOT` 和 `DEER_FLOW_CONFIG_PATH` 可以避免启动目录或旧环境变量导致 Gateway 读取错误的配置文件。

在另一个终端检查健康状态：

```bash
curl http://localhost:8001/health
```

Gateway 直连地址为 `http://localhost:8001`。

## 2. 启动 Frontend

打开第二个终端：

```bash
export DEER_FLOW_PROJECT_ROOT=/home/glory/code/github/deer-flow
cd "$DEER_FLOW_PROJECT_ROOT/frontend"
pnpm run dev
```

Frontend 地址为 `http://localhost:3000`。

Next.js 默认只加载 `frontend/` 下的 `.env*` 文件，不会自动加载仓库根目录的 `.env`。如果前端服务确实需要根目录中的非敏感环境变量，可在启动前显式加载：

```bash
export DEER_FLOW_PROJECT_ROOT=/home/glory/code/github/deer-flow
set -a
source "$DEER_FLOW_PROJECT_ROOT/.env"
set +a
cd "$DEER_FLOW_PROJECT_ROOT/frontend"
pnpm run dev
```

不要把密钥放入 `NEXT_PUBLIC_*` 变量，因为这类变量会被编译进浏览器代码。

### 不使用 Nginx 时直连 Gateway

如果只启动 Gateway 和 Frontend，可指定浏览器直接访问 Gateway：

```bash
export DEER_FLOW_PROJECT_ROOT=/home/glory/code/github/deer-flow
cd "$DEER_FLOW_PROJECT_ROOT/frontend"

NEXT_PUBLIC_BACKEND_BASE_URL=http://localhost:8001 \
NEXT_PUBLIC_LANGGRAPH_BASE_URL=http://localhost:8001/api \
pnpm run dev
```

修改 `NEXT_PUBLIC_*` 后需要重启前端开发服务器。

### 使用 Nginx 时

如果使用下一节的统一入口，不要设置上述两个 `NEXT_PUBLIC_*` 变量。浏览器请求将通过 Nginx 转发。

## 3. 启动 Nginx

确认 Gateway 和 Frontend 已经启动，再打开第三个终端：

```bash
export DEER_FLOW_PROJECT_ROOT=/home/glory/code/github/deer-flow
cd "$DEER_FLOW_PROJECT_ROOT"

mkdir -p \
  logs \
  temp/{client_body_temp,proxy_temp,fastcgi_temp,uwsgi_temp,scgi_temp}

nginx -g 'daemon off;' \
  -c "$DEER_FLOW_PROJECT_ROOT/docker/nginx/nginx.local.conf" \
  -p "$DEER_FLOW_PROJECT_ROOT"
```

统一访问地址：

```text
http://localhost:2026
```

本地 Nginx 会将页面请求转发到 Frontend 的 `3000` 端口，将 API 请求转发到 Gateway 的 `8001` 端口。

## 停止服务

各组件均以前台方式运行。在对应终端按 `Ctrl+C` 即可停止。

如果 Nginx 已转入后台或残留了旧进程，可先检查配置和进程，再按本机运维规范停止；不要直接结束系统中其他项目使用的 Nginx 实例。

## 环境变量加载说明

- Gateway 的应用代码通常能向上找到仓库根目录 `.env`，但本文仍显式 `source`，以减少工作目录和调试器环境带来的差异。
- Frontend 直接运行 `pnpm dev` 时不会向父目录查找 `.env`；它只自动加载 `frontend/.env*`。
- 独立打开的新终端不会继承另一个终端中的环境变量，因此每个终端都应设置自己需要的变量。
- `source .env` 前使用 `set -a`，可确保文件中定义的变量被导出给后续子进程。
- Shell 中已经导出的变量通常优先于应用通过 `load_dotenv(override=False)` 加载的同名变量。

## 常见问题

### Gateway 没有读取预期的配置

检查路径变量，但不要输出密钥值：

```bash
printf 'project root: %s\n' "$DEER_FLOW_PROJECT_ROOT"
printf 'config path: %s\n' "$DEER_FLOW_CONFIG_PATH"
test -f "$DEER_FLOW_CONFIG_PATH" && echo 'config exists' || echo 'config missing'
```

### 模型环境变量没有加载

只检查变量是否存在，不要打印实际值：

```bash
if [ -n "${OPENAI_API_KEY:-}" ]; then
  echo 'OPENAI_API_KEY is set'
else
  echo 'OPENAI_API_KEY is missing'
fi
```

### 端口被占用

检查默认端口：

```bash
ss -ltnp | grep -E ':(8001|3000|2026)\b'
```

默认端口对应关系：

| 组件 | 端口 |
| --- | ---: |
| Gateway | 8001 |
| Frontend | 3000 |
| Nginx | 2026 |

### 是否需要启动 Provisioner

普通本地沙箱模式不需要。只有 `config.yaml` 使用 `AioSandboxProvider` 并配置了 `provisioner_url` 时，才需要额外部署 Provisioner/Kubernetes 运行环境。