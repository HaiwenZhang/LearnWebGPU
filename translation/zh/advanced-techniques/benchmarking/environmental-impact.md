环境影响（<span class="bullet">🔴</span>TODO）
====================

```{translation-warning} 译文可能已过时, /advanced-techniques/benchmarking/environmental-impact.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

如果我们希望能够与其他（可能非基于软件的）解决方案进行比较来解决应用程序所解决的问题，那么能够至少粗略地评估应用程序对环境的影响非常重要。

GPU 密集型应用程序会通过多种费用对环境产生影响，例如：

- 使用应用程序时能源消耗的影响取决于计算运行的时间、用户在典型工作流程中使用密集计算的时间以及计算的强度。

- 最低要求限制的选择会影响制造新硬件的需求。

用于测量**即时能量**费用的工具在很大程度上取决于平台。例如，NVidia 通过 [NVML 库](https://developer.nvidia.com/nvidia-management-library-nvml) 提供信息。

这种能源的**碳影响**取决于用户的位置。有几个 API 可以查询此信息，例如 [WattTime](https://www.watttime.org/api-documentation).

**制造业影响**更难以评估，特别是对于非碳相关领域，其中尤其包括具有挑战性的不可再生的**稀有金属**的提取。
