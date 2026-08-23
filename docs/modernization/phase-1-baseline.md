# Phase 1: 可审计基线

## 目标

在改变架构或增加产品功能前，把 Mirage 的现有行为变成可重复验证的证据。Phase 1 完成后，每个重构提交都必须能够回答：

1. 哪些既有行为被测试覆盖？
2. 哪些行为仍依赖人工冒烟测试？
3. 本次修改改变了什么，为什么属于预期变化？
4. Intel 与 Apple Silicon 是否得到同等验证？
5. 发布产物的源码、版本、签名和依赖是否可以追溯？

测试降低回归风险，但不单独构成安全保证。安全结论还需要权限边界、输入验证、依赖审计、代码签名、公证和威胁模型共同支撑。

## 冻结点

- 上游版本：`v1.0.9`
- 上游提交：`afb4af7b568d822b8b42aee388be48cc946a02b3`
- 改造分支：`architecture-baseline`
- `main` 暂时保留为上游镜像，不在 Phase 1 直接开发。

`v1.0.9` 之后的私有动态锁屏扩展不进入基线。屏保/Web 渲染稳定性修复 `b0253cb` 等基础门禁建立后单独审计，不与架构整理混合提交。

## 工具链纪律

- Xcode 27 beta 用于开发期兼容验证。
- TestFlight、App Store Connect、Developer ID 和 notarization 产物只允许使用正式版 Xcode。
- CI 与本地验证必须记录 Xcode、SDK、编译器、架构和 Git commit。
- beta 与正式版 Xcode 使用不同 DerivedData，避免模块缓存相互污染。

当前 Mirage 不满足 Mac App Store/TestFlight 的基本形态：主 App 未启用 App Sandbox，并会启动独立 renderer、管理外部 Wallpaper Engine 内容。Phase 1 不承诺 TestFlight；直接分发的 Developer ID + notarization 是近期可实现路径。是否建立 App Store-compatible 变体留到安全架构审查后决定。

## 门禁层级

### Gate A：快速回归

- SceneRenderer 现有 5 个 regression tests 必须全部被发现并通过。
- 测试使用独立 CMake build tree，不污染产品构建缓存。
- Apple Silicon 与 Intel CI 都运行同一套测试。
- CI 必须在无发布密钥、无 Steam API key 的环境中运行。

### Gate B：组件构建与协议

- 三个 renderer 分别构建。
- SteamService 完成 build、启动、`hello`、`ping`、`shutdown` IPC smoke test。
- 为 renderer JSON-lines IPC 建立 schema/contract fixtures。
- 对 malformed JSON、迟到事件、进程提前退出和队列背压增加测试。

### Gate C：Swift 主 App

- 建立真实的 Swift test target，删除失效的旧测试引用。
- 将路径 containment、播放策略合并、显示器标识、playlist 选择等纯逻辑移出 UI 生命周期后测试。
- 对 `RendererController` 先写 characterization tests，再拆分职责。

### Gate D：行为与安全

- 建立不包含第三方受版权保护内容的最小 scene/web/video fixtures。
- 验证启动恢复、切换 first-frame、失败 rollback、暂停/恢复、显示器移除等状态转换。
- Web 壁纸覆盖本地文件边界、外部网络、导航、弹窗和持久化行为。
- 对依赖、entitlements、嵌套可执行文件和发布包生成审计报告。

### Gate E：分发

- 所有嵌套可执行文件使用同一 Developer Team 的适当证书签名。
- Developer ID 直发包启用 Hardened Runtime、secure timestamp，并通过 notarization 与 Gatekeeper 验证。
- Sparkle 更新签名和 Apple code signing 分开管理、分别验证。
- TestFlight 只有在 App Store sandbox/public API 路线被证明可行后启用。

## 第二阶段的进入条件

满足以下条件后，才开始拆分 `RendererController` 和 `WorkshopViewModel`：

- Gate A 在两个架构上稳定通过。
- Gate B 已覆盖 renderer 构建与 Steam IPC。
- Swift test target 真实执行，日志中能证明测试数量大于零。
- 已记录至少一轮本地或 CI 全量构建基线。
- 每个待拆模块都有职责清单、状态所有权和回滚方案。
- 尚未自动化的 GUI/TCC/多显示器场景进入人工冒烟清单。

## Phase 1 首批工作

- [x] 从 `v1.0.9` 建立 `architecture-baseline`。
- [x] 新增独立 SceneRenderer test runner。
- [x] 新增 Intel + Apple Silicon regression CI。
- [ ] 修复 Xcode Scheme 中失效的旧测试引用。
- [ ] 建立 Swift test target 与第一批纯逻辑 characterization tests。
- [ ] 把 SteamService IPC smoke test 从发布 workflow 提取成无密钥质量门禁。
- [ ] 建立三种最小壁纸 fixture 的许可证与来源清单。
- [ ] 记录正式 Xcode 与 Xcode beta 的本地构建基线。
- [ ] 设计 Developer ID/notarization 流程；证书与私钥只保存在 Apple/钥匙串/CI secret 中。

## 审计记录格式

每个 Phase 1 PR 或 commit 至少记录：

- 改动范围与非目标
- 对应风险
- 自动测试名称与真实执行数量
- 人工验证项目
- 两种架构的结果
- 使用的 Xcode/SDK/compiler 版本
- 已知未覆盖项
- 是否改变发布、权限或数据格式
