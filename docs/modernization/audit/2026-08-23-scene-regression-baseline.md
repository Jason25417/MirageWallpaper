# SceneRenderer 回归基线（2026-08-23）

## 范围

这是 `v1.0.9` 架构冻结点的第一份可重复验证记录，只覆盖 SceneRenderer 已存在的 5 个 regression tests。它不代表完整 App、Steam IPC、签名、公证或最低系统版本兼容性已经通过。

## 来源与环境

- Git commit：`afb4af7b568d822b8b42aee388be48cc946a02b3`
- 主机：macOS 27.0 beta（`26A5416b`），Apple Silicon
- 发布工具链：Xcode 26.6（`17F113`）
- macOS SDK：26.5，明确来自正式版 Xcode
- CMake：4.4.2
- C/C++ compiler：Homebrew Clang 22.1.8
- CMake preset：`macos-arm64-clang-release`
- deployment target：macOS 14.2

## 执行结果

命令：

```bash
DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer \
  ./SceneRenderer/scripts/test.sh
```

结果：5 个测试被发现，5 个通过。

- `SceneRendererTextGeometry`
- `SceneRendererLayerVisibility`
- `SceneRendererRuntimeCompatibility`
- `SceneRendererScriptCompatibility`
- `SceneRendererPuppetAlpha`

## 门禁发现的问题

第一次运行虽然设置了 `DEVELOPER_DIR`，CMake 仍从独立安装的 Command Line Tools 缓存了 macOS 27 SDK framework 路径，导致 Metal header 编译失败。test runner 现已明确传入 `CMAKE_OSX_SYSROOT`，并按 SDK 名称隔离 build tree，避免正式版与 beta SDK 的 CMake cache 互相污染。

链接阶段还发现本机 Homebrew 的 Vulkan、FFmpeg、LZ4、FreeType 和 Fontconfig dylib 是按 macOS 26.0 或更高版本构建的，而项目声明 deployment target 为 14.2。因此本次结果只能证明当前主机上的 arm64 regression tests 通过，不能证明产物能在 macOS 14.2 运行。

## 未覆盖与后续动作

- Intel 结果等待 GitHub Actions 验证。
- Apple Silicon CI 结果等待 GitHub Actions 验证。
- 需要把 release dependencies 固定为可追溯、与最低系统版本一致的构建产物。
- 需要建立 Swift test target、renderer 构建门禁和 Steam IPC smoke tests。
- 本次没有修改产品行为、权限、数据格式或发布签名。
