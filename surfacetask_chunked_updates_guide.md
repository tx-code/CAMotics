# SurfaceTask 分块更新与最终合并实现指南

## 当前架构概述

### SurfaceTask 类结构
```cpp
class SurfaceTask : public Task {
    cb::SmartPointer<SimulationRun> simRun;
    cb::SmartPointer<Surface> surface;
public:
    void run() override;  // 当前一次性处理整个表面
};
```

### 当前工作流程
1. `SurfaceTask::run()` → `simRun->compute(task)` 
2. `SimulationRun::compute()` 创建 ToolSweep 和 GridTree
3. `Renderer::render()` 一次性渲染整个工件
4. 从 GridTree 提取最终表面

## 分块更新策略

### 方案1: 空间分块 (Spatial Chunking)

#### 实现思路
将3D工作空间按照xyz轴分割成多个子区域，每个子区域独立处理。

```cpp
class ChunkedSurfaceTask : public SurfaceTask {
private:
    struct Chunk {
        cb::Rectangle3D bounds;
        cb::SmartPointer<GridTree> tree;
        cb::SmartPointer<Surface> surface;
        bool completed = false;
    };
    
    std::vector<Chunk> chunks;
    cb::SmartPointer<Surface> mergedSurface;
    
public:
    void run() override;
    void processChunk(size_t chunkIndex);
    cb::SmartPointer<Surface> mergeChunks();
};
```

#### 具体实现步骤

1. **初始化分块**
```cpp
void ChunkedSurfaceTask::initializeChunks() {
    Rectangle3D totalBounds = simRun->getSimulation().workpiece.getBounds();
    
    // 按分辨率和内存限制计算最优分块数
    Vector3D chunkSize = calculateOptimalChunkSize(totalBounds);
    
    for (int x = 0; x < chunkCountX; x++) {
        for (int y = 0; y < chunkCountY; y++) {
            for (int z = 0; z < chunkCountZ; z++) {
                Chunk chunk;
                chunk.bounds = calculateChunkBounds(x, y, z, chunkSize);
                chunks.push_back(chunk);
            }
        }
    }
}
```

2. **处理单个分块**
```cpp
void ChunkedSurfaceTask::processChunk(size_t chunkIndex) {
    Chunk& chunk = chunks[chunkIndex];
    
    // 为当前分块创建局部GridTree
    chunk.tree = new GridTree(Grid(chunk.bounds, simRun->getSimulation().resolution));
    
    // 创建局部CutWorkpiece (只包含影响此区域的工具路径)
    auto localSweep = createLocalToolSweep(chunk.bounds);
    CutWorkpiece cutWP(localSweep, simRun->getSimulation().workpiece);
    
    // 渲染当前分块
    Renderer renderer(*this);
    renderer.render(cutWP, *chunk.tree, chunk.bounds, 1, sim.mode);
    
    // 提取分块表面
    chunk.surface = new TriangleSurface(*chunk.tree);
    chunk.completed = true;
    
    // 更新总体进度
    updateProgress();
}
```

3. **进度更新**
```cpp
void ChunkedSurfaceTask::updateProgress() {
    size_t completed = 0;
    for (const auto& chunk : chunks) {
        if (chunk.completed) completed++;
    }
    
    double progress = (double)completed / chunks.size();
    update(progress);
}
```

4. **最终合并**
```cpp
cb::SmartPointer<Surface> ChunkedSurfaceTask::mergeChunks() {
    auto mergedSurface = new TriangleSurface();
    
    for (const auto& chunk : chunks) {
        if (chunk.surface && chunk.completed) {
            // 合并三角形网格
            mergedSurface->merge(*chunk.surface);
        }
    }
    
    // 处理分块边界的接缝
    mergedSurface->resolveSeams();
    
    return mergedSurface;
}
```

### 方案2: 时间分块 (Temporal Chunking)

#### 实现思路
按时间段分割工具路径，逐步构建表面。

```cpp
class TemporalChunkedSurfaceTask : public SurfaceTask {
private:
    struct TimeChunk {
        double startTime;
        double endTime;
        cb::SmartPointer<ToolSweep> sweep;
        bool processed = false;
    };
    
    std::vector<TimeChunk> timeChunks;
    cb::SmartPointer<GridTree> accumulatedTree;
    
public:
    void run() override;
    void processTimeChunk(size_t chunkIndex);
};
```

#### 实现步骤

1. **时间分割**
```cpp
void TemporalChunkedSurfaceTask::initializeTimeChunks() {
    double totalTime = simRun->getSimulation().path->getTime();
    double chunkDuration = calculateOptimalChunkDuration();
    
    for (double t = 0; t < totalTime; t += chunkDuration) {
        TimeChunk chunk;
        chunk.startTime = t;
        chunk.endTime = std::min(t + chunkDuration, totalTime);
        timeChunks.push_back(chunk);
    }
}
```

2. **逐步处理**
```cpp
void TemporalChunkedSurfaceTask::processTimeChunk(size_t chunkIndex) {
    TimeChunk& chunk = timeChunks[chunkIndex];
    
    // 创建时间段内的工具扫描
    chunk.sweep = new ToolSweep(simRun->getSimulation().path, 
                               chunk.startTime, chunk.endTime);
    
    // 更新累积的GridTree
    CutWorkpiece cutWP(chunk.sweep, simRun->getSimulation().workpiece);
    
    // 增量式渲染到累积树中
    Renderer renderer(*this);
    Rectangle3D bbox = chunk.sweep->getBounds().grow(sim.resolution * 1.1);
    renderer.render(cutWP, *accumulatedTree, bbox, sim.threads, sim.mode);
    
    chunk.processed = true;
    
    // 发布中间结果 (可选)
    publishIntermediateResult();
    
    updateProgress();
}
```

### 方案3: 混合分块 (Hybrid Chunking)

结合空间和时间分块，提供最大灵活性：

```cpp
class HybridChunkedSurfaceTask : public SurfaceTask {
private:
    struct HybridChunk {
        cb::Rectangle3D spatialBounds;
        double startTime, endTime;
        cb::SmartPointer<Surface> surface;
        bool completed = false;
    };
    
    std::vector<std::vector<HybridChunk>> chunkGrid; // [time][space]
    
public:
    void run() override;
    void processHybridChunk(size_t timeIndex, size_t spaceIndex);
    cb::SmartPointer<Surface> mergeAll();
};
```

## 任务管理集成

### 与 ConcurrentTaskManager 集成

```cpp
class ChunkedSurfaceTaskManager {
public:
    void startChunkedSurfaceTask(const Simulation& sim) {
        auto mainTask = new ChunkedSurfaceTask(sim);
        
        // 创建子任务
        for (size_t i = 0; i < mainTask->getChunkCount(); i++) {
            auto chunkTask = new ChunkProcessingTask(mainTask, i);
            taskManager.addTask(chunkTask, false); // 低优先级
        }
        
        // 添加合并任务
        auto mergeTask = new MergeTask(mainTask);
        taskManager.addTask(mergeTask, true); // 高优先级，等待所有分块完成
    }
};
```

### 进度聚合

```cpp
class ChunkProgressAggregator : public TaskObserver {
    std::vector<double> chunkProgress;
    
public:
    void onTaskProgress(Task* task, double progress) override {
        if (auto chunkTask = dynamic_cast<ChunkProcessingTask*>(task)) {
            chunkProgress[chunkTask->getChunkIndex()] = progress;
            
            double totalProgress = std::accumulate(chunkProgress.begin(), 
                                                 chunkProgress.end(), 0.0) 
                                 / chunkProgress.size();
            
            notifyMainTask(totalProgress);
        }
    }
};
```

## 内存管理策略

### 动态分块大小调整
```cpp
Vector3D calculateOptimalChunkSize(const Rectangle3D& bounds) {
    size_t availableMemory = getAvailableMemory();
    double memoryPerChunk = availableMemory * MEMORY_USAGE_RATIO;
    
    // 根据网格分辨率和内存限制计算分块大小
    double resolution = simRun->getSimulation().resolution;
    double voxelsPerChunk = memoryPerChunk / sizeof(GridPoint);
    double sideLength = cbrt(voxelsPerChunk) * resolution;
    
    return Vector3D(sideLength, sideLength, sideLength);
}
```

### 分块生命周期管理
```cpp
class ChunkLifecycleManager {
    std::queue<cb::SmartPointer<Surface>> completedChunks;
    size_t maxCachedChunks = 5;
    
public:
    void addCompletedChunk(cb::SmartPointer<Surface> surface) {
        completedChunks.push(surface);
        
        // 内存压力下清理早期分块
        if (completedChunks.size() > maxCachedChunks) {
            completedChunks.pop();
        }
    }
};
```

## 中间结果发布

### 实时预览支持
```cpp
class IntermediateResultPublisher {
public:
    void publishChunkResult(const Chunk& chunk, double overallProgress) {
        // 创建部分表面预览
        auto previewSurface = createPreviewSurface(chunk);
        
        // 通知UI更新
        emit surfaceChunkCompleted(previewSurface, overallProgress);
    }
    
private:
    cb::SmartPointer<Surface> createPreviewSurface(const Chunk& chunk) {
        // 创建低精度预览或边界框表示
        return chunk.surface->createLowResPreview();
    }
};
```

## 错误处理与恢复

### 分块失败处理
```cpp
void ChunkedSurfaceTask::handleChunkFailure(size_t chunkIndex, const std::exception& e) {
    LOG_WARNING("Chunk " << chunkIndex << " failed: " << e.what());
    
    // 重试策略
    if (chunks[chunkIndex].retryCount < MAX_RETRIES) {
        chunks[chunkIndex].retryCount++;
        retryChunk(chunkIndex);
    } else {
        // 使用降级策略或跳过该分块
        handleChunkSkip(chunkIndex);
    }
}
```

## 使用示例

```cpp
// 在 QtWin.cpp 中使用
void QtWin::startChunkedSurface() {
    auto chunkedTask = new ChunkedSurfaceTask(sim);
    
    // 配置分块参数
    chunkedTask->setChunkingStrategy(ChunkingStrategy::SPATIAL);
    chunkedTask->setMaxChunkSize(Vector3D(50, 50, 50)); // mm
    chunkedTask->enableIntermediatePreview(true);
    
    // 设置进度回调
    chunkedTask->setProgressCallback([this](double progress, const Surface& intermediate) {
        updateProgress(progress);
        updatePreview(intermediate);
    });
    
    taskMan.addTask(chunkedTask);
}
```

## 性能优化建议

1. **自适应分块**: 根据几何复杂度动态调整分块大小
2. **并行处理**: 在多核系统上并行处理独立分块
3. **内存映射**: 对大型数据集使用内存映射文件
4. **增量合并**: 在分块完成时逐步合并，而非等待全部完成
5. **缓存策略**: 智能缓存常用分块，减少重复计算

这种分块更新与最终合并的方法可以显著改善大型工件的处理性能，提供更好的用户体验和内存使用效率。