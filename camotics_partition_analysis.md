# CAMotics仿真模块分区管理及更新机制分析

## 概述

CAMotics是一个开源的CAM（计算机辅助制造）仿真软件，其仿真模块采用了多层次的空间分区策略来高效处理3D切削仿真。该系统主要通过网格分区、八叉树和并行渲染来实现高性能的切削仿真计算。

## 核心组件架构

### 1. 空间分区体系结构

#### 1.1 Grid（网格）系统
- **位置**: `src/camotics/Grid.h/cpp`
- **功能**: 基础的3D空间网格划分
- **关键特性**:
  - 使用`Vector3D offset`、`double resolution`和`Vector3U steps`定义3D网格
  - 支持递归分区：`partition()`方法可将网格按指定数量分割
  - 智能分割策略：优先按最大维度分割（`largestDim()`）
  - 切片操作：`slice()`方法支持创建子网格

```cpp
void Grid::partition(vector<Grid> &grids, unsigned count) const {
  if (count < 2) {
    grids.push_back(*this);
    return;
  }
  // 按最大维度分割
  pair<Grid, Grid> parts = split(largestDim());
  unsigned half = count / 2;
  parts.first.partition(grids, half);
  parts.second.partition(grids, count - half);
}
```

#### 1.2 GridTree（网格树）系统
- **位置**: `src/camotics/contour/GridTree.h/cpp`
- **功能**: 分层的网格管理系统
- **结构特点**:
  - 继承自`GridTreeNode`和`Grid`
  - 支持动态分区：`partition()`方法考虑边界框交集
  - 叶子节点插入：`insertLeaf()`管理几何数据

#### 1.3 OctTree（八叉树）系统
- **位置**: `src/camotics/sim/OctTree.h/cpp`
- **功能**: 工具路径的空间索引
- **特性**:
  - 8个子节点的递归结构
  - 移动查找优化：`MoveLookup`接口
  - 碰撞检测：`collisions()`方法快速查找相交的移动指令

### 2. 分区管理策略

#### 2.1 自适应分区算法
网格分区采用递归二分法：
1. **维度选择**: 选择最大维度进行分割
2. **负载均衡**: 分割数量按比例分配
3. **边界处理**: 考虑实际工作边界框的交集

#### 2.2 多线程并行处理
- **位置**: `src/camotics/render/Renderer.cpp`
- **策略**: 
  - 目标作业数 = `2^(ceil(log(threads)/log(2)) + 2)`
  - 动态负载分配
  - 实时进度跟踪

```cpp
// 分割工作负载
unsigned targetJobCount = pow(2, ceil(log(threads) / log(2)) + 2);
tree.partition(jobGrids, bbox, targetJobCount);

// 并行执行渲染作业
while (!jobGrids.empty() && jobs.size() < threads) {
  SmartPointer<RenderJob> job = 
    new RenderJob(*this, cutWorkpiece, mode, jobGrids.back());
  job->start();
  jobs.push_back(job);
  jobGrids.pop_back();
}
```

## 更新机制

### 1. 增量更新策略

#### 1.1 时间范围更新
- **位置**: `src/camotics/sim/SimulationRun.cpp`
- **机制**: 
  - 首次计算建立完整扫描（`ToolSweep`）
  - 后续更新仅处理时间变化范围
  - 使用`setChange()`方法标记变更区域

```cpp
if (sweep.isNull()) {
  // 首次完整扫描
  sweep = new ToolSweep(sim.path);
  bbox = sim.workpiece.getBounds().grow(sim.resolution * 0.9);
  tree = new GridTree(Grid(bbox, sim.resolution));
} else {
  // 增量更新
  double minTime = simTime;
  double maxTime = simTime;
  if (lastTime < minTime) minTime = lastTime;
  if (maxTime < lastTime) maxTime = lastTime;
  
  SmartPointer<MoveLookup> change = new ToolSweep(sim.path, minTime, maxTime);
  sweep->setChange(change);
  bbox = change->getBounds().grow(sim.resolution * 1.1);
}
```

#### 1.2 空间区域更新
- 仅更新受影响的边界框区域
- 动态调整边界框大小（增长1.1倍分辨率作为缓冲）

### 2. 数据结构更新

#### 2.1 GridTreeNode更新机制
- **位置**: `src/camotics/contour/GridTreeNode.cpp`
- **过程**:
  1. 递归插入叶子节点
  2. 动态创建子节点
  3. 实时更新节点计数

```cpp
void GridTreeNode::insertLeaf(GridTreeLeaf *leaf, const Vector3U &_steps,
                              const Vector3U &offset) {
  // 左分支或右分支插入
  if (offset[axis] < split) {
    if (atLeaf(steps)) {
      if (left) delete left;
      left = leaf;
    } else {
      if (!left) left = new GridTreeNode(steps);
      left->insertLeaf(leaf, steps, offset);
    }
  } else {
    // 右分支处理逻辑
  }
  
  // 更新节点统计
  count = (left ? left->getCount() : 0) + (right ? right->getCount() : 0);
}
```

#### 2.2 三角网格更新
- **组件**: `GridTreeLeaf`存储三角形数据
- **更新**: 通过`add(Triangle)`方法动态添加几何数据
- **优化**: 使用Marching Cubes或Corrected MC33算法生成表面

## 渲染管道

### 1. 表面生成流程

#### 1.1 FieldFunction计算
- 基于工具扫描和工件材料计算距离场
- 支持多种工具形状（球形、圆锥形等）

#### 1.2 等值面提取
- **算法**: Marching Cubes / Cubical Marching Squares
- **优化**: MC33修正算法处理歧义情况
- **并行化**: 每个网格分区独立处理

### 2. 任务调度机制

#### 2.1 RenderJob系统
- **位置**: `src/camotics/render/RenderJob.cpp`
- **特点**:
  - 每个作业处理一个GridTreeRef
  - 支持中断机制
  - 条件变量同步

#### 2.2 进度跟踪
- 实时计算完成百分比
- 估算剩余时间（ETA）
- 支持任务取消

## 性能优化策略

### 1. 内存管理
- 智能指针管理（`SmartPointer`）
- 延迟创建子节点
- 自动清理未使用的分区

### 2. 计算优化
- 边界框预检查避免不必要计算
- 缓存工具扫描结果
- 多线程并行处理

### 3. 数据局部性
- 分区按空间邻近性组织
- 减少缓存未命中
- 优化内存访问模式

## 总结

CAMotics的仿真模块通过以下关键技术实现高效的分区管理：

1. **分层分区**: Grid → GridTree → GridTreeNode → GridTreeLeaf
2. **自适应策略**: 根据数据特征和硬件资源动态调整分区
3. **增量更新**: 仅处理时间和空间上的变化区域
4. **并行处理**: 多线程渲染作业提高计算效率
5. **智能缓存**: 复用计算结果，避免重复工作

这种设计既保证了仿真精度，又实现了良好的性能和可扩展性，特别适合大规模复杂零件的切削仿真应用。