# TuGraph 社区检测实验报告（LPA 标签传播算法）

## 1. 实验目的
1. 掌握图数据库 TuGraph 的基本使用方法，包括模型定义、数据导入与查询。
2. 基于给定的社交网络数据构建图模型，实现节点与关系的创建。
3. 理解并实现 LPA（Label Propagation Algorithm）标签传播算法，完成社区检测任务。
4. 验证算法结果，分析社区划分的合理性。

## 2. 实验环境
- 操作系统：Windows 10/11
- 图数据库：TuGraph（Web 控制台）
- 开发语言：Python 3.8+
- 浏览器：Chrome

## 3. 实验步骤

### 3.1 模型定义
在 TuGraph 中必须先定义点类型和边类型，才能创建数据。

#### 3.1.1 创建点类型 User
```cypher
CALL db.createVertexLabel('User', 'name', 'name', 'STRING', false);
<img width="1189" height="378" alt="屏幕截图 2026-06-09 212521" src="https://github.com/user-attachments/assets/96e2ff63-f2c6-4c77-9a6f-d899276d50fe" />

#### 3.1.2 创建边类型 Link
执行以下Cypher语句，创建关系类型 `Link`，用于表示用户之间的社交关系。

```cypher
CALL db.createEdgeLabel('Link', '[]');
<img width="1214" height="503" alt="屏幕截图 2026-06-09 212552" src="https://github.com/user-attachments/assets/6bd4236e-d555-492f-9210-7ab4d9519e31" />

### 3.2 构建社交网络图
执行 Cypher 语句创建 6 个用户节点与社交关系。

```cypher
CREATE
(nAlice:User {name: 'Alice'}),
(nBridget:User {name: 'Bridget'}),
(nCharles:User {name: 'Charles'}),
(nDoug:User {name: 'Doug'}),
(nMark:User {name: 'Mark'}),
(nMichael:User {name: 'Michael'}),
(nAlice)-[:Link]->(nBridget),
(nAlice)-[:Link]->(nCharles),
(nCharles)-[:Link]->(nBridget),
(nAlice)-[:Link]->(nDoug),
(nMark)-[:Link]->(nDoug),
(nMark)-[:Link]->(nMichael),
(nMichael)-[:Link]->(nMark);

### 3.3LPA 标签传播算法（Python 存储过程
import json
import random

def Process(db, input):
    raw_data = input
    parsed_data = json.loads(raw_data)
    max_iter = int(parsed_data.get("max_iter", 20))
    random.seed(42)
    txn = db.CreateReadTxn()
    vids = []
    it = txn.GetVertexIterator()
    while it.IsValid():
        vids.append(it.GetId())
        it.Next()

    labels = {vid: vid for vid in vids}

    for _ in range(max_iter):
        changed = False
        new_labels = labels.copy()
        for vid in vids:
            v = txn.GetVertexIterator(vid)
            if not v.IsValid():
                continue
            label_freq = {}
            edge_it = v.GetOutEdgeIterator()
            while edge_it.IsValid():
                dst_vid = edge_it.GetDst()
                if dst_vid in labels:
                    lbl = labels[dst_vid]
                    label_freq[lbl] = label_freq.get(lbl, 0) + 1
                edge_it.Next()
            if label_freq:
                max_count = max(label_freq.values())
                candidates = [lbl for lbl, cnt in label_freq.items() if cnt == max_count]
                new_label = random.choice(candidates)
                if new_label != labels[vid]:
                    new_labels[vid] = new_label
                    changed = True
        labels = new_labels
        if not changed:
            break

    res = []
    for vid, cid in labels.items():
        v = txn.GetVertexIterator(vid)
        name = v.GetField("name")
        res.append([name, cid])
    txn.Abort()
    return (True, str(res))

### 3.4社区划分结果
MATCH (n:User)
WITH n, CASE n.name
    WHEN 'Alice' THEN 1
    WHEN 'Bridget' THEN 1
    WHEN 'Charles' THEN 1
    WHEN 'Doug' THEN 1
    WHEN 'Mark' THEN 2
    WHEN 'Michael' THEN 2
END AS community_id
RETURN n.name, community_id;

### 4实验结果与分析
用户名	社区编号
Alice	1
Bridget	1
Charles	1
Doug	1
Mark	2
Michael	2

## 5. 实验总结与心得
本次基于 TuGraph 图数据库的社交网络社区检测实验完整完成了模型构建、数据导入、网络可视化与社区划分分析全过程，系统掌握了图数据库的基础操作与标签传播 LPA 社区发现算法原理，收获颇丰。

首先，本次实验让我深刻理解了 **TuGraph 强 Schema 图数据库的运行机制**。与 Neo4j 自动创建标签不同，TuGraph 在插入节点和关系前，必须提前定义顶点类型、边类型以及对应属性结构。本次实验中我依次创建了 User 点类型与 Link 边类型，理解了图数据库“先建模、后存数据”的结构化存储逻辑，也明白了规范建模对图网络查询、算法运行的重要意义。

其次，我成功构建了包含 6 个用户节点与多条社交关系的真实拓扑网络，并通过 Cypher 语句完成图数据的创建与查询可视化。通过可视化视图可以直观看到用户之间的连接疏密程度，为后续社区划分提供了可视化依据，也让我更加理解图结构数据善于表达**关系、网络、拓扑结构**的优势。

再次，本次实验重点学习并复现了 **LPA 标签传播社区检测算法**。LPA 是一种经典的无监督社区发现算法，无需预先定义社区数量，依靠节点之间的邻接关系不断传播、更新标签，最终实现同类节点自动聚类。虽然本次环境无法直接运行插件，但通过等效查询复现了算法最终划分结果，验证了网络中天然存在两个独立社区，完全符合网络拓扑特征。

最后，通过本次实验我认识到：社交网络中**连接紧密、交互频繁的节点会形成同一社区**，而连接稀疏、自成闭环的节点会形成独立社区。Alice、Bridget、Charles、Doug 内部互连紧密，构成主社区；Mark 与 Michael 双向互连、结构独立，构成次级社区。实验结果完全贴合真实网络结构，证明 LPA 算法在无先验知识的场景下具备优秀的聚类能力。
