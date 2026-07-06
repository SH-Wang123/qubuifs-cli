# QubuiFS-CLI 设计需求

## 目标

QubuiFS-CLI 是 QubuiFS 的命令行管理与测试工具。当前阶段先从管理面开始测试，围绕控制面 REST 端点完成连通性验证、接口契约验证、资源操作验证和故障输出归一化。

首版不实现数据面压测或 FUSE 操作辅助，只面向管理面 OpenAPI 中已经暴露的接口。

## 基本原则

- [ ] 命令行统一使用 `qubuifs` 命名，不使用旧项目名。
- [ ] 所有管理面请求基于 `openapi/openapi.yaml` 对齐参数和响应结构。
- [ ] CLI 不把控制面业务错误包装成成功。
- [ ] 对后端尚未实现的接口，要明确显示 `not implemented` 或“后端未实现”。
- [ ] 默认输出适合人读；支持 JSON 输出用于自动化测试。

## 配置模型

CLI 需要支持三类配置来源，优先级从高到低：

1. 命令行参数，例如 `--endpoint http://127.0.0.1:9665`。
2. 环境变量，例如 `QUBUIFS_ENDPOINT=http://127.0.0.1:9665`。
3. 本地配置文件，例如 `$HOME/.config/qubuifs/cli.yaml`。

基础配置项：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `endpoint` | 无 | 控制面 REST 端点，必填或通过 `connect` 写入配置 |
| `timeout` | `10s` | 单次管理面请求超时 |
| `output` | `table` | 输出格式，支持 `table`、`json` |
| `insecure` | `false` | HTTPS 场景下是否跳过证书校验 |

## 管理面测试范围

首轮测试只覆盖管理面：

- [ ] 控制面存活检查：`GET /healthz`。
- [ ] 控制面就绪检查：`GET /readyz`。
- [ ] 控制面版本读取：`GET /version`。
- [ ] QubuiFS 列表、创建、查询、删除。
- [ ] 节点列表、添加、查询、删除、状态读取。
- [ ] 节点设备列表、添加设备、删除设备。
- [ ] QubuiFS 拓扑节点注册和反注册。
- [ ] 元数据导出。

当前后端仍存在未完成语义，CLI 测试需要按真实结果报告，不强行判定所有接口必须成功。

## 命令结构

建议命令分组：

```text
qubuifs
  connect
  status
  test management
  fs list|create|get|delete
  node list|add|get|delete|status
  device list|add|delete
  topology register|unregister
  metadata export
```

### 连接命令

`qubuifs connect` 用于验证并保存控制面端点。

示例：

```bash
qubuifs connect --endpoint http://127.0.0.1:9665
```

验收要求：

- [ ] 校验 endpoint 格式，要求包含协议和 host。
- [ ] 请求 `GET /healthz`，要求 `code=0` 且 `data.status=ok`。
- [ ] 请求 `GET /version`，失败时只标记版本未知。
- [ ] 请求 `GET /readyz`，不以 `ready=false` 阻断连接。
- [ ] 成功后写入本地配置文件。

### 状态命令

`qubuifs status` 展示当前控制面状态。

输出内容：

- [ ] endpoint。
- [ ] healthz 状态。
- [ ] readyz 状态和原因。
- [ ] version、git commit、build time，如果控制面返回。

### 管理面测试命令

`qubuifs test management` 是当前优先级最高的命令，用于跑一组管理面 API 测试。

示例：

```bash
qubuifs test management --endpoint http://127.0.0.1:9665 --output json
```

测试分级：

- `smoke`：只测 `healthz`、`version`、`readyz`。
- `readonly`：在 `smoke` 基础上测试 list/get/status 类接口。
- `write`：执行 create/add/delete/export 等写操作，需要显式传 `--write`。

默认只跑 `smoke` 和安全的只读检查，避免误删或误建资源。

## 测试结果模型

每个测试项输出统一结构：

```json
{
  "name": "healthz",
  "method": "GET",
  "path": "/healthz",
  "status": "passed",
  "http_status": 200,
  "code": 0,
  "message": "success",
  "duration_ms": 12
}
```

`status` 取值：

- `passed`：接口符合预期。
- `failed`：接口返回错误或响应结构不符合预期。
- `skipped`：缺少必要参数或当前测试等级不执行。
- `not_implemented`：控制面明确返回未实现。

## 已知接口状态处理

CLI 需要认识当前管理面状态：

- [ ] `healthz` 可作为强握手条件。
- [ ] `readyz` 目前可能返回 `ready=false`，不能作为首版阻断条件。
- [ ] `version` 目前可能只有 `version=dev`，缺少构建信息时只提示缺失。
- [ ] 节点列表、节点查询、节点删除、节点状态、节点设备列表可能返回未实现。
- [ ] QubuiFS 拓扑注册/反注册可能返回未实现。
- [ ] 数据设备增删依赖后端节点 runtime 是否完整建立，失败时输出真实错误。

## 输出规范

人读输出示例：

```text
Endpoint: http://127.0.0.1:9665

PASS healthz                  200  success
WARN readyz                   200  ready=false reason=not implemented
PASS version                  200  version=dev
SKIP node.list                not implemented
```

JSON 输出必须保持稳定字段，便于 CI 使用。

## 容器与分发

CLI 自身可以作为二进制分发，也可以提供测试容器。

测试容器目标：

- [ ] 镜像名使用 `qubuifs` 风格，例如 `harbor.local/qubuifs/qubuifs-cli:latest`。
- [ ] 容器启动后默认执行 `qubuifs test management`。
- [ ] 通过环境变量传入控制面端点。

示例：

```bash
docker run --rm \
  -e QUBUIFS_ENDPOINT=http://10.0.0.12:9665 \
  harbor.local/qubuifs/qubuifs-cli:latest \
  test management --output json
```

## 里程碑

### M1. 文档与契约

- [x] 同步管理面 OpenAPI 文档。
- [x] 编写 CLI 管理面测试设计需求。
- [ ] 标注每个 OpenAPI operation 的测试等级和当前后端状态。

### M2. CLI 骨架

- [ ] 初始化 CLI 工程。
- [ ] 实现配置加载和 endpoint 解析。
- [ ] 实现 OpenAPI 响应基础模型。

### M3. 管理面测试

- [ ] 实现 `connect`。
- [ ] 实现 `status`。
- [ ] 实现 `test management --level smoke`。
- [ ] 实现 `test management --level readonly`。
- [ ] 实现写操作测试的显式开关。

### M4. 容器化

- [ ] 增加容器构建文件。
- [ ] 增加容器运行示例。
- [ ] 在真实控制面端点上完成 smoke 测试。
