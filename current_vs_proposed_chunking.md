# CAMotics 当前机制 vs 提议的分块方案

## 当前已存在的并行机制

### 1. GridTree 级别的空间分割 (已实现)
```cpp
// 在 Renderer::render() 中
unsigned targetJobCount = pow(2, ceil(log(threads) / log(2)) + 2);
tree.partition(jobGrids, bbox, targetJobCount);  // 分割网格空间

// 创建并行RenderJob
while (!jobGrids.empty() && jobs.size() < threads) {
    SmartPointer<RenderJob> job = new RenderJob(*this, cutWorkpiece, mode, jobGrids.back());
    job->start();  // 在单独线程中运行
    jobs.push_back(job);
}
```

**特点**：
- ✅ **已实现** - 这是现有的渲染并行化
- 🔄 **运行时分块** - 在单个SurfaceTask执行期间进行
- 📊 **网格级别** - 分割的是GridTree的3D网格空间  
- ⚡ **内存中操作** - 所有分块同时在内存中处理
- 🚫 **无中间结果** - 用户看不到渐进式更新

### 2. 任务级别的并发 (已实现)
```cpp
// ConcurrentTaskManager 管理多个Task
ConcurrentTaskManager taskMan;
taskMan.addTask(new SurfaceTask(sim));
taskMan.addTask(new ToolPathTask(sim));  // 可以并行运行不同类型的任务
```

## 我提议的分块方案 (未实现)

### 与现有机制的关键区别

| 方面 | 现有机制 | 提议的分块方案 |
|------|----------|----------------|
| **分块层次** | GridTree渲染层 | SurfaceTask任务层 |
| **内存使用** | 全部数据同时在内存 | 按需加载，可释放完成的分块 |
| **用户体验** | 一次性完成，无中间反馈 | 渐进式更新，实时预览 |
| **可中断性** | 当前Task内部可中断 | 可在分块边界优雅中断 |
| **错误恢复** | 整个任务失败需重启 | 单个分块失败可重试 |
| **并行范围** | 单个Surface的网格并行 | 多个Surface分块可真正并行 |

### 提议方案的三种策略详解

#### 1. 空间分块 (Surface-Level Spatial Chunking)
```cpp
// 这是我提议的，目前不存在
class ChunkedSurfaceTask : public SurfaceTask {
    std::vector<Chunk> chunks;  // 每个chunk是独立的Surface任务
    
    void run() override {
        // 创建多个独立的SurfaceTask，每个处理一个空间区域
        for (auto& chunk : chunks) {
            auto chunkTask = new SingleChunkSurfaceTask(chunk.bounds);
            taskManager.addTask(chunkTask);
        }
        // 等待全部完成后合并
    }
};
```

**与现有GridTree分割的区别**：
- 🆕 **任务级分块** vs 网格级分块
- 💾 **可序列化** - 分块结果可保存到磁盘
- 🔄 **可恢复** - 可从中断点继续
- 👁️ **用户可见** - 每个分块完成时显示进度

#### 2. 时间分块 (Temporal Chunking)  
```cpp
// 完全新的概念，当前不存在
class TemporalChunkedSurfaceTask : public SurfaceTask {
    void processTimeSegment(double startTime, double endTime) {
        // 只处理这个时间段的工具路径
        auto partialPath = fullPath->getSegment(startTime, endTime);
        // 逐步更新同一个Surface
    }
};
```

#### 3. 混合分块 (Hybrid Chunking)
```cpp
// 结合空间+时间的新方法
class HybridChunkedSurfaceTask : public SurfaceTask {
    // 既按空间分割，又按时间分段
    std::vector<std::vector<HybridChunk>> chunkGrid; // [time][space]
};
```

## 实现路径

### 短期：增强现有机制
```cpp
// 在现有Renderer基础上添加中间结果发布
void Renderer::render(...) {
    // ... 现有代码 ...
    
    // 新增：发布中间结果
    if (completedJobs % PREVIEW_INTERVAL == 0) {
        auto intermediateSurface = tree.extractPartialSurface();
        task.publishIntermediate(intermediateSurface);
    }
}
```

### 长期：实现任务级分块
```cpp
// 全新的分块任务类
class ChunkedSurfaceTask : public SurfaceTask {
    // 完整的分块、合并、进度管理逻辑
};
```

## 总结

**现有的并行化**：
- ✅ 已有GridTree级别的空间分割并行渲染
- ✅ 已有任务管理器支持多任务并发
- ❌ 但用户体验仍是"全有或全无"

**提议的分块化**：  
- 🆕 任务级别的分块处理
- 🆕 渐进式结果展示
- 🆕 更好的内存管理
- 🆕 更强的错误恢复能力

这两种方法是**互补的**，不是替代关系。提议的方案可以在现有并行渲染基础上，提供更高层次的分块管理。