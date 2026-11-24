# ADK JS S3ArtifactService 独立实施计划

状态：已确定施工方向，尚未实现。执行仓库为 rong-coder/adk-js（google/adk-js 的 fork），本文件为独立贡献计划和 Codex 工作指令。不得自动提交上游 Issue/PR。实施分支从干净的 upstream 基线创建，贡献计划文件不进入上游 PR。

## 0. 目标与边界

实现通用、可上游合并的 S3ArtifactService，符合 Google ADK JS 的 BaseArtifactService 契约；支持 AWS S3 及经实测的 S3 兼容服务（my-superagent 优先验收 Garage，另验收 MinIO）。不依赖 my-superagent 的 Registry、Inbox、Task、Workspace、Artifact Gateway 或数据库。上游 PR 合并不是 S1 的前置条件；在合并前可使用经测试的 fork 构建，锁定 commit/包版本。此任务不修改 Runner、Workflow、Rehydration，也不解决崩溃恢复 Issue。

基线：my-superagent 已验证 @google/adk@2.5.0，对应 gitHead b1b94c0d501c3d1eb028aa413d783d6573f6c785。开始时确认 fork 与 upstream/main 实际状态，搜索已有 S3 实现/开放 PR，避免重复工作；在最新合适上游基线建独立分支供 PR，若与 2.5.0 不兼容则明确记录并单独适配锁定版本，不把上游不相关提交混入 PR。不得自动升级 my-superagent 的 ADK 版本。

## 1. 阅读真实接口与仓库规范

- [BaseArtifactService](https://github.com/google/adk-js/blob/b1b94c0d501c3d1eb028aa413d783d6573f6c785/core/src/artifacts/base_artifact_service.ts)
- [GcsArtifactService](https://github.com/google/adk-js/blob/b1b94c0d501c3d1eb028aa413d783d6573f6c785/core/src/artifacts/gcs_artifact_service.ts)：可选 peer 动态加载、Part 序列化、版本接口。
- [InMemoryArtifactService](https://github.com/google/adk-js/blob/b1b94c0d501c3d1eb028aa413d783d6573f6c785/core/src/artifacts/in_memory_artifact_service.ts)：作用域与可逆 key 编码。
- [共享 Artifact 测试](https://github.com/google/adk-js/blob/b1b94c0d501c3d1eb028aa413d783d6573f6c785/core/test/artifacts/artifact_service_test_utils.ts)；[GCS 测试](https://github.com/google/adk-js/blob/b1b94c0d501c3d1eb028aa413d783d6573f6c785/core/test/artifacts/gcs_artifact_service_test.ts)。
- [core/package.json](https://github.com/google/adk-js/blob/b1b94c0d501c3d1eb028aa413d783d6573f6c785/core/package.json)、core/src/index.ts、index_web.ts、core/build.js、core/src/artifacts/registry.ts、core/src/utils/optional_peer.ts、实际 devtools 入口及包发布测试。
- [CONTRIBUTING.md](https://github.com/google/adk-js/blob/main/CONTRIBUTING.md)：npm workspaces、依赖声明、Conventional Commits、CLA、人工审查 AI 草稿。

实际执行以目标分支接口为准。复用现有 helper 和测试模式；可参考其他语言 S3 实现，但须先核实许可证、接口与并发语义，不复制未经授权代码。

## 2. 公共 API

建议新增 core/src/artifacts/s3_artifact_service.ts，导出 S3ArtifactService；沿用 GCS 风格的构造函数，例如：

```ts
import { S3ArtifactService } from '@google/adk/artifacts/s3';

const service = new S3ArtifactService('artifact-bucket', {
  region: 'us-east-1',
  endpoint: 'http://127.0.0.1:3900',
  forcePathStyle: true,
});
```

以上是拟实现 API，不是当前已存在接口。构造参数建议为 bucket 与可选 SDK S3ClientConfig（如需 storage prefix，使用独立适配器选项，不将其误传 SDK）。AWS 默认凭据链、region、endpoint、forcePathStyle、自定义凭据由 SDK 管理；支持私有桶和 TLS，不默认公开对象，不关闭签名或证书验证。不在 URI、日志、canonicalUri 中暴露凭据。存储桶由部署者预创建，不隐式创建或改变桶策略。

| 原生方法（2.5.0） | 行为 |
| --- | --- |
| saveArtifact(request): Promise<number> | 写入 Part，返回从 0 开始递增的逻辑版本 |
| loadArtifact(request): Promise<Part \| undefined> | 按指定版本或最新版本加载；真正不存在返回 undefined |
| listArtifactKeys(request): Promise<string[]> | 返回当前 Session 与同一用户共享作用域的文件名，去重排序 |
| deleteArtifact(request): Promise<void> | 删除该文件名的全部逻辑版本；不存在时幂等成功 |
| listVersions(request): Promise<number[]> | 升序返回所有可见逻辑版本 |
| listArtifactVersions(request): Promise<ArtifactVersion[]> | 按版本返回 canonicalUri、mimeType、customMetadata |
| getArtifactVersion(request): Promise<ArtifactVersion \| undefined> | 获取指定或最新版本元数据 |

不改变 BaseArtifactService 请求结构、返回值或现有存储实现。通过 Runner 原生 artifactService 注入，不依赖自建服务。

## 3. 存储模型与序列化

- 使用可逆、无碰撞的分段编码，区分 session 与 user scope。示例：<prefix>/session/<app>/<user>/<session>/<encoded filename>/<version>，用户共享 user: 文件名使用 <prefix>/user/<app>/<user>/<encoded filename>/<version>。各动态段独立编码（包含斜线、百分号、Unicode、点段等）；严格校验长度、版本整数与完整前缀；不将用户共享目录与名为 user 的 Session 混淆。文件名带斜线是逻辑名，不是访问其他命名空间的路径。
- 一个逻辑版本优先存为一个原子提交对象，避免字节已写但元数据未写或反之。设计格式须文档化并版本化。可选方案：单对象存实际 bytes + 受限且可逆的元数据编码，或单对象封装 Part 与元数据；若采用 S3 user metadata，核实 header 大小/字符限制、内部字段与自定义字段冲突，并对超限或不可序列化输入明确报错；若采用封装格式，说明额外空间/读取开销并评估 HeadObject 取 metadata 的能力。不要默默截断或丢失字段。
- Part 至少覆盖现有实现的 text（含空文本、Unicode）、inlineData（二进制/base64、mimeType、displayName）和 fileData（fileUri、mimeType 等支持字段），fileData 只保存引用，不自动下载任意 URL。保留文本与 text/plain 二进制的区分；无效 Part 或非法 base64 明确拒绝。自定义 metadata 不修改调用方对象，保留 JSON 可表达类型或遵循共享测试明确要求的字符串语义，记录选定契约并覆盖 number/boolean/nested/null、保留键冲突。不要宣称支持所有 GenAI Part 类型。
- canonicalUri 为稳定的 s3://bucket/key 标识，不意味着公开可访问或预签名授权；通过 SDK 加载。逻辑版本不依赖 S3 bucket versioning（Garage 不一定支持），与 my-superagent Workspace 的 Git 历史无关。
- 不将 ETag 当通用内容哈希（multipart、加密等情况下不成立）；依靠 SDK/后端支持的校验机制，实测字节往返和截断/错误响应；必要时使用 SHA-256 校验，记录协议与失败行为。Part API 本身面向完整对象，不宣称无限大文件流式传输；my-superagent 大型 Workspace/Artifact 的独立流式 Gateway 不在本 PR。

## 4. 并发、错误与兼容性（重点）

不能直接复制 GCS 的 listVersions + max+1 + 无条件覆盖，也不能把进程内 mutex 宣称为跨进程安全。

1. 首选以已验证原子条件写入为版本分配原语：列出现有逻辑版本，候选为 max+1（空集为 0）；PutObject 使用 IfNoneMatch: '*' 创建候选对象。412/明确条件冲突时重新列举与有限退避重试；409 按 SDK/后端实际冲突语义重试；禁止无条件 Put 覆盖已存在版本。保存成功后才返回版本。并行独立 client/process 测试证明同一版本只被分配一次。
2. 请求超时或响应丢失可能是写入成功但结果未知。每次 save 内部使用唯一写入标识（写在同一对象元数据/封装中），对账候选版本及标识；确认同一次写入已成功可返回该版本，确认被其他写入者占用才申请新版本，无法确认则显式报错，不盲重试造成额外版本。BaseArtifactService 没有调用方 idempotencyKey，不承诺跨任意应用重试 exactly-once；my-superagent 的业务幂等另由 Manager 负责。
3. AWS S3 条件写入是明确能力，不等于所有兼容后端都支持。查阅 [AWS conditional writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html) 和 [Garage S3 compatibility](https://github.com/deuxfleurs-org/garage/blob/main-v2/doc/book/reference-manual/s3-compatibility.md)，实测目标 Garage/MinIO 版本的条件创建、并发冲突及 list 可见性。必须包含“同一 key 第二次条件写入应失败且原内容不变”的探针。若后端不具备原子条件创建或等效保证，不可静默退化为不安全多写者；报告兼容缺口，明确单写者限制及需要调用方串行化的部署方式（my-superagent 可在中央 Artifact Gateway 保证串行），或由用户批准其他方案。不得为通用 PR 强加 SQL/Redis 分布式锁或私有业务依赖。只有实际通过验证的模式才能标注为支持。
4. DeleteArtifact 按精确文件前缀枚举并删除全部版本，处理分页及部分失败；删除后再次保存的版本从 0 开始，遵守原契约。并发 save/delete 不可假定 S3 多对象事务；文档规定调用方对同一 artifact 的删除与写入进行排他协调，验收受支持范围内的行为，不夸大线性一致性。若上游要求更强保证，另行讨论。
5. ListObjectsV2 处理 continuation token、空页、千级对象、严格数字后缀、去重与数字排序；listArtifactKeys 不把内部记录或相邻文件名前缀误识为文件。批量删除处理 API 上限及每对象错误；listArtifactVersions 的 metadata 获取控制并发。
6. 仅明确的 NoSuchKey/对象 404 视为不存在；AccessDenied、无效凭据、网络故障、损坏内容、无桶等错误不能统统吞掉成 undefined。错误不含 secret、签名 URL 或 payload；有限重试、可观察失败，不引入自定义重试风暴。

## 5. 上游集成与实现顺序

| 工作单元 | 交付 |
| --- | --- |
| P0 | 记录目标分支/commit、上游重复检查、API 差异、真实后端能力探针、存储格式/并发契约 |
| P1 | key codec、Part codec、metadata、S3 client 初始化、原子版本分配与读写 |
| P2 | 全部七个接口、分页、删除、错误分类与故障对账 |
| P3 | 导出、构建及依赖声明、URI registry（若适合）、文档与独立示例 |
| P4 | 单测、真实 S3 兼容集成、打包/Runner 验证、回归与证据 |
| P5 | 人工 review，整理上游 PR 说明；用户确认后提交 PR |

参照 GCS：AWS SDK v3 @aws-sdk/client-s3 在 core 中声明为 optional peer 与开发测试依赖（遵守目标分支规范），动态 import + loadOptionalPeer；import type 不引入运行时加载。遵循实际 build entry、ESM/CJS、types、package exports 及 Node-only/browser 隔离。考虑根导出与 @google/adk/artifacts/s3 子路径；检查 esbuild external 及真实打包产物。未安装 SDK 的普通 ADK 用户仍可正常 import/运行，只有实际选择 S3 时才给出明确安装提示。不得让 browser bundle 引入 Node/AWS 运行时。若添加 s3://bucket 形式的 registry 支持，先核实 registry 的同步/异步构造与凭据传递方式；endpoint 等设置不塞入带 secret 的 URI。CLI/config 自动发现若需要大范围变动，作为后续独立 PR，而不是阻塞核心类。

## 6. 验收矩阵

- 复用 runArtifactServiceTests；新增 Vitest 单测覆盖七方法、0/1/2 版本、latest、missing、删除后重建、跨 Session 的 user: 共享与 app/user 隔离、同前缀文件、特殊字符/Unicode、text/inlineData/fileData、metadata 往返及不修改原对象。
- SDK mock 必须模拟真实条件失败与分页，不仅使用内存 Map 的无条件覆盖。测试并行 32 个 save（多个独立实例）、412/409、超时已提交/未提交、无法对账、403、404、损坏数据、分页、部分删除失败、并发重试耗尽。
- 真实后端：锁定 Garage、MinIO 镜像/版本，用临时测试桶和非生产凭据；执行跨进程 32 并发写入同一 artifact，验证版本唯一、内容正确、最新版本、原子条件探针、分页/清理、重启后读取。AWS S3 若有获授权测试账号则执行同一套测试；无账号则标记未实测，不虚报支持。记录各后端真实能力与配置（包括 region/path style/checksum 策略），不把 mock 当真实验证。若 Garage 无法满足条件写入，明确首版中央单写者运行契约和限制，保留安全失败测试。
- Runner 最小集成测试：实际注入 S3ArtifactService，保存/加载后新进程或新实例读回；不引入真实 LLM 费用。打包 tarball 后独立 consumer 测 ESM/CJS/types、SDK 未安装情况下 import、S3 缺依赖提示与 browser 构建不回归。
- 以 fork 实际脚本为准，基线已有 npm run build、npm test、npm run lint、npm run format:check、npm run ts:check、npm run docs:check；先跑 npm ci（有匹配 lockfile 时），定向 Vitest 后跑全量门禁。记录真实命令/结果；外部服务不可用时如实列出未跑测试及风险，不标记全部通过。不要修改 CI 来跳过失败门禁。

## 7. 上游 PR 与 my-superagent 接入

贡献按 [官方 CONTRIBUTING.md](https://github.com/google/adk-js/blob/main/CONTRIBUTING.md)；Apache-2.0 文件头遵循邻近文件及上游约定，依赖许可证核实，Google CLA 由用户签署，AI 初稿须经人工审查和完善。PR 标题：`feat(artifacts): add S3-compatible artifact service`（Conventional Commits，100 字符内，无句号）。PR 说明范围、七接口、配置示例、兼容矩阵、并发设计/限制、测试证据及不改变既有实现。保留精简 diff，不提交凭据、私有项目链接、无关重构或真实用户数据。不自动发 PR；用户决定最终提交。

my-superagent S1 只通过通用 BaseArtifactService 接入；本地/远程权限、Workspace 发布与清理、Skill 索引、流式 Gateway 仍由自身实现。上游不接受 PR 时继续使用锁定并验收的 fork 版本；上游合并后经过接口/持久数据格式兼容、并发与恢复回归才切换。交付物包括实际 commit、独立测试记录、兼容矩阵、包构建结果和 PR 文案。
