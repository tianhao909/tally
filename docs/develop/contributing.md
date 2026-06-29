# Contributing to Tally

Thank you for your interest in contributing to Tally! This guide helps you get started.

## Getting Started

1. Fork the repository on GitHub
2. Clone your fork locally:
   ```bash
   git clone --recursive https://github.com/<your-username>/tally.git
   cd tally
   ```
3. Create a feature branch:
   ```bash
   git checkout -b feature/your-feature-name
   ```

## Development Setup

### Build with Debug Options

```bash
mkdir -p build && cd build
cmake .. -DENABLE_LOGGING=ON -DENABLE_PERFORMANCE_LOGGING=ON -DVERIFY_CORRECTNESS=ON
make -j$(nproc)
```

### Running Tests

```bash
./scripts/run_test.sh
```

## Code Organization

- `src/tally/` — Core C++ implementation
- `include/tally/` — Public headers
- `python/tally/` — Code generation tools
- `scripts/` — Build and runtime scripts
- `tests/` — Test suite
- `third_party/` — External dependencies

## Contribution Types

### Bug Fixes
- Create an issue describing the bug
- Reference the issue in your PR
- Include a test case if possible

### New Scheduling Policies
- Add implementation in `src/tally/scheduler/`
- Register the policy in `src/tally/env.cpp`
- Add documentation in `docs/scheduling-policies.md`

### New Device Support
- Extend the CUDA API coverage in `python/tally/preload/`
- Add message structures in `include/tally/msg_struct.h`
- Regenerate client code

### Documentation
- Keep English and Chinese versions in sync
- Update both README.md and README_cn.md for user-facing changes

## Code Style

- C++20 standard
- 4 spaces indentation
- Use `snake_case` for functions and variables
- Use `CamelCase` for class names
- Use `UPPER_CASE` for constants and macros

## Pull Request Process

1. Ensure your code compiles without warnings
2. Run the test suite
3. Update documentation if needed
4. Submit PR against the `main` branch
5. Describe what your PR does and why

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

# 贡献指南

感谢您对 Tally 项目的贡献兴趣！本指南帮助您快速上手。

## 开始

1. 在 GitHub 上 Fork 仓库
2. 克隆您的 Fork：
   ```bash
   git clone --recursive https://github.com/<your-username>/tally.git
   cd tally
   ```
3. 创建功能分支：
   ```bash
   git checkout -b feature/your-feature-name
   ```

## 开发环境

### 带调试选项构建

```bash
mkdir -p build && cd build
cmake .. -DENABLE_LOGGING=ON -DENABLE_PERFORMANCE_LOGGING=ON -DVERIFY_CORRECTNESS=ON
make -j$(nproc)
```

### 运行测试

```bash
./scripts/run_test.sh
```

## 贡献类型

- **Bug 修复**：创建 Issue 描述问题，在 PR 中引用
- **新调度策略**：在 `src/tally/scheduler/` 中添加实现
- **新设备支持**：扩展 `python/tally/preload/` 中的 CUDA API 覆盖
- **文档**：保持中英文版本同步

## 代码风格

- C++20 标准
- 4 空格缩进
- 函数和变量使用 `snake_case`
- 类名使用 `CamelCase`
- 常量和宏使用 `UPPER_CASE`

## 许可证

贡献代码将在 MIT 许可证下发布。
