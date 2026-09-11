---
'@juejin-opensource/jusage-core': patch
'@juejin-opensource/jusage': patch
---

修复 Windows 上的三处失效：Desktop 与 CLI 的「谁占用本地服务」判定因依赖已从 Windows 11 移除的 `wmic` 而始终失败，两端可能同时把自己当成 owner；`AI_USAGE_*_ROOTS` 系列环境变量会在盘符冒号处被切断，导致 Kilo Code / Roo Code 等数据源读不到任何用量；并发写入 `config.json` 时可能因文件被占用而报 EPERM 丢失一次写入。
