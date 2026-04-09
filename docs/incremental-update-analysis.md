# Claude Context 增量更新功能分析

本文档分析 Claude Context 项目中的增量更新（Incremental Update）功能的实现原理和架构。

## 目录

- [功能概述](#功能概述)
- [架构设计](#架构设计)
- [核心组件](#核心组件)
- [同步策略](#同步策略)
- [代码位置](#代码位置)
- [改进建议](#改进建议)

---

## 功能概述

Claude Context 已经实现了完整的增量更新功能，基于 **Merkle DAG** 数据结构来检测文件变化，并自动在后台定期同步。

### 核心能力

- ✅ **文件变化检测**：基于 SHA-256 哈希和 Merkle DAG 快速检测新增、删除、修改的文件
- ✅ **增量向量更新**：只重新索引变化的文件，不需要完全重建索引
- ✅ **后台自动同步**：MCP 服务器启动后自动开启定期同步
- ✅ **快照持久化**：文件哈希状态持久化存储，重启后可继续增量检测

---

## 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                       MCP Server                            │
│  ┌────────────────────────────────────────────────────────┐│
│  │                    SyncManager                          ││
│  │  • startBackgroundSync() - 启动后台同步                 ││
│  │  • handleSyncIndex() - 执行同步逻辑                     ││
│  │  • 初始延迟 5 秒后首次同步                              ││
│  │  • 每 5 分钟自动同步一次                                ││
│  └────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Context.reindexByChange()                   │
│  ┌────────────────────────────────────────────────────────┐│
│  │ 1. FileSynchronizer.checkForChanges()                  ││
│  │    - 基于 Merkle DAG 检测文件变化                       ││
│  │    - 返回 {added, removed, modified}                   ││
│  │                                                         ││
│  │ 2. 删除 removed 文件的向量                              ││
│  │ 3. 删除 modified 文件的旧向量                           ││
│  │ 4. 重新索引 added + modified 文件                       ││
│  └────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    FileSynchronizer                         │
│  ┌────────────────────────────────────────────────────────┐│
│  │ • 基于 SHA-256 文件哈希                                 ││
│  │ • Merkle DAG 结构快速检测变化                           ││
│  │ • 快照存储在 ~/.context/merkle/{hash}.json             ││
│  └────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## 核心组件

### 1. SyncManager（同步管理器）

**文件位置**: `packages/mcp/src/sync.ts`

负责管理后台同步任务的调度：

```typescript
export class SyncManager {
    private context: Context;
    private snapshotManager: SnapshotManager;
    private isSyncing: boolean = false;

    // 启动后台同步
    public startBackgroundSync(): void {
        // 初始延迟 5 秒后首次同步
        setTimeout(async () => {
            await this.handleSyncIndex();
        }, 5000);

        // 每 5 分钟定期同步
        setInterval(() => {
            this.handleSyncIndex();
        }, 5 * 60 * 1000);
    }

    // 执行同步逻辑
    public async handleSyncIndex(): Promise<void> {
        const indexedCodebases = this.snapshotManager.getIndexedCodebases();
        
        for (const codebasePath of indexedCodebases) {
            const stats = await this.context.reindexByChange(codebasePath);
            console.log(`Sync complete: Added: ${stats.added}, Removed: ${stats.removed}, Modified: ${stats.modified}`);
        }
    }
}
```

### 2. FileSynchronizer（文件同步器）

**文件位置**: `packages/core/src/sync/synchronizer.ts`

负责检测文件系统变化：

```typescript
export class FileSynchronizer {
    private fileHashes: Map<string, string>;  // 文件路径 -> SHA-256 哈希
    private merkleDAG: MerkleDAG;             // Merkle DAG 结构
    private snapshotPath: string;             // 快照存储路径

    // 检测文件变化
    public async checkForChanges(): Promise<{
        added: string[],
        removed: string[],
        modified: string[]
    }> {
        // 1. 生成当前文件系统的哈希
        const newFileHashes = await this.generateFileHashes(this.rootDir);
        
        // 2. 构建新的 Merkle DAG
        const newMerkleDAG = this.buildMerkleDAG(newFileHashes);
        
        // 3. 比较 DAG 检测变化
        const changes = MerkleDAG.compare(this.merkleDAG, newMerkleDAG);
        
        // 4. 更新状态并保存快照
        this.fileHashes = newFileHashes;
        this.merkleDAG = newMerkleDAG;
        await this.saveSnapshot();
        
        return changes;
    }
}
```

### 3. MerkleDAG（Merkle 有向无环图）

**文件位置**: `packages/core/src/sync/merkle.ts`

用于高效比较文件集合的变化：

```typescript
export class MerkleDAG {
    nodes: Map<string, MerkleDAGNode>;
    rootIds: string[];

    // 比较两个 DAG
    public static compare(dag1: MerkleDAG, dag2: MerkleDAG): {
        added: string[],
        removed: string[],
        modified: string[]
    } {
        const nodes1 = new Map(dag1.getAllNodes().map(n => [n.id, n]));
        const nodes2 = new Map(dag2.getAllNodes().map(n => [n.id, n]));

        const added = Array.from(nodes2.keys()).filter(k => !nodes1.has(k));
        const removed = Array.from(nodes1.keys()).filter(k => !nodes2.has(k));
        
        return { added, removed, modified };
    }
}
```

### 4. Context.reindexByChange()（增量重建索引）

**文件位置**: `packages/core/src/context.ts`

核心增量更新逻辑：

```typescript
async reindexByChange(codebasePath: string): Promise<{
    added: number,
    removed: number,
    modified: number
}> {
    // 1. 检测文件变化
    const { added, removed, modified } = await synchronizer.checkForChanges();

    // 2. 删除已移除文件的向量
    for (const file of removed) {
        await this.deleteFileChunks(collectionName, file);
    }

    // 3. 删除已修改文件的旧向量
    for (const file of modified) {
        await this.deleteFileChunks(collectionName, file);
    }

    // 4. 重新索引新增和修改的文件
    const filesToIndex = [...added, ...modified];
    await this.processFileList(filesToIndex, codebasePath);

    return { added: added.length, removed: removed.length, modified: modified.length };
}
```

---

## 同步策略

### 当前同步时机

| 触发时机 | 说明 |
|---------|------|
| MCP 服务器启动后 5 秒 | 首次同步，检测离线期间的文件变化 |
| 每 5 分钟 | 定期后台同步 |

### 快照存储

快照文件存储在用户主目录下：

```
~/.context/merkle/{md5_hash}.json
```

其中 `{md5_hash}` 是代码库绝对路径的 MD5 哈希值。

快照内容包括：
- `fileHashes`: 所有文件的 SHA-256 哈希映射
- `merkleDAG`: 序列化的 Merkle DAG 结构

---

## 代码位置

| 文件 | 功能 |
|------|------|
| `packages/mcp/src/sync.ts` | **SyncManager** - 后台定时同步管理器 |
| `packages/mcp/src/index.ts` | MCP 服务器入口，初始化 SyncManager |
| `packages/core/src/sync/synchronizer.ts` | **FileSynchronizer** - 文件变化检测 |
| `packages/core/src/sync/merkle.ts` | **MerkleDAG** - 增量检测数据结构 |
| `packages/core/src/context.ts` | **reindexByChange()** - 增量更新核心逻辑 |

---

## 改进建议

### 可能的改进方向

| 改进项 | 工作量 | 说明 |
|--------|--------|------|
| **缩短同步间隔** | ⭐ 简单 (5分钟) | 修改 `sync.ts` 中的定时器间隔 |
| **添加手动同步 MCP 工具** | ⭐⭐ 中等 (30分钟) | 添加 `sync_index` 工具到 handlers |
| **文件系统 Watcher** | ⭐⭐⭐ 复杂 (2-4小时) | 使用 `chokidar` 监听文件变化，实时触发同步 |
| **增量同步优化** | ⭐⭐⭐⭐ 较复杂 (1天) | 优化大量文件变化时的批量处理性能 |

### 快速调整同步间隔

如果需要更频繁的同步，可以修改 `packages/mcp/src/sync.ts`：

```typescript
// 原来：每 5 分钟
setInterval(() => {
    this.handleSyncIndex();
}, 5 * 60 * 1000);

// 修改为：每 1 分钟
setInterval(() => {
    this.handleSyncIndex();
}, 1 * 60 * 1000);
```

### 添加手动同步工具

在 `packages/mcp/src/handlers.ts` 中添加：

```typescript
public async handleSyncIndex(args: any) {
    const { path: codebasePath } = args;
    const absolutePath = ensureAbsolutePath(codebasePath);
    
    const stats = await this.context.reindexByChange(absolutePath);
    
    return {
        content: [{
            type: "text",
            text: `Sync completed for '${absolutePath}'.\nAdded: ${stats.added}, Removed: ${stats.removed}, Modified: ${stats.modified}`
        }]
    };
}
```

### 实时文件监听（高级）

使用 `chokidar` 实现文件系统事件监听：

```typescript
import chokidar from 'chokidar';

class RealtimeSyncManager {
    private watcher: chokidar.FSWatcher | null = null;
    private pendingChanges: Set<string> = new Set();
    private debounceTimer: NodeJS.Timeout | null = null;

    public startWatching(codebasePath: string): void {
        this.watcher = chokidar.watch(codebasePath, {
            ignored: this.ignorePatterns,
            persistent: true,
            ignoreInitial: true
        });

        this.watcher
            .on('add', path => this.scheduleSync(path))
            .on('change', path => this.scheduleSync(path))
            .on('unlink', path => this.scheduleSync(path));
    }

    private scheduleSync(filePath: string): void {
        this.pendingChanges.add(filePath);
        
        // 防抖：等待 2 秒后批量处理
        if (this.debounceTimer) {
            clearTimeout(this.debounceTimer);
        }
        
        this.debounceTimer = setTimeout(() => {
            this.processPendingChanges();
        }, 2000);
    }

    private async processPendingChanges(): Promise<void> {
        const changes = Array.from(this.pendingChanges);
        this.pendingChanges.clear();
        
        await this.context.reindexByChange(this.codebasePath);
    }
}
```

---

## 总结

Claude Context 的增量更新功能已经相当完善，核心组件包括：

1. **SyncManager** - 后台同步调度
2. **FileSynchronizer** - 基于 Merkle DAG 的文件变化检测
3. **Context.reindexByChange()** - 增量向量更新

当前默认每 5 分钟自动同步一次。如果需要更实时的更新，可以缩短同步间隔或实现文件系统 Watcher。
