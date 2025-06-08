# 大模型的图社区检测及生成

> 基于 [neo4j的官方demo](https://github.com/neo4j-labs/llm-graph-builder)的0.8.2分支。
>
> 主要用于社区的检测和社区节点创建的逻辑梳理。

在图论中，社区是图的一组节点，这些节点之间的连接比图中其他节点的连接更紧密。

简单来说，就是找出图中关系更紧密的节点群组「子图」。

让我们看一个具体的例子：

```text
假设我们有以下实体：
- 实体A：Python编程语言
- 实体B：Django框架
- 实体C：Flask框架
- 实体D：JavaScript
- 实体E：React
- 实体F：Vue

这些实体之间有关系：
Python -> Django (使用)
Python -> Flask (使用)
JavaScript -> React (使用)
JavaScript -> Vue (使用)
```

在这个例子中，自然会形成两个社区：

- 社区1：Python、Django、Flask（Python生态圈）

- 社区2：JavaScript、React、Vue（JavaScript生态圈）

社区检测的主要目的是为了：

- 将大规模知识图谱分解成更小的、语义相关的子图

- 提供多层次的知识组织结构

- 优化知识检索和问答效果



是社区检测与生成的过程。

> 前置条件，我们已经生成了Document、Chunk，并通过大模型提取出了entity和响应的relationship。
>
> ```cypher
> // Document节点结构
> (:Document {
>     // 基本信息
>     fileName: String,          // 文件名（主键）
>     fileSize: Integer,         // 文件大小
>     fileType: String,         // 文件类型（如 "pdf", "txt" 等）
>     fileSource: String,       // 文件来源（如 "local", "web-url", "gcs" 等）
>     url: String,              // URL（针对web来源）
>     language: String,         // 文档语言
>     
>     // 状态信息
>     status: String,           // 处理状态（"New", "Failed", "Cancelled" 等）
>     errorMessage: String,     // 错误信息
>     is_cancelled: Boolean,    // 是否被取消
>     retry_condition: String,  // 重试条件
>     
>     // 时间信息
>     createdAt: DateTime,      // 创建时间
>     updatedAt: DateTime,      // 更新时间
>     processingTime: Float,    // 处理耗时
>     
>     // 统计信息
>     nodeCount: Integer,           // 总节点数
>     relationshipCount: Integer,   // 总关系数
>     total_chunks: Integer,        // 总块数
>     processed_chunk: Integer,     // 已处理块数
>     chunkNodeCount: Integer,      // 块节点数
>     chunkRelCount: Integer,       // 块关系数
>     entityNodeCount: Integer,     // 实体节点数
>     entityEntityRelCount: Integer,// 实体间关系数
>     communityNodeCount: Integer,  // 社区节点数
>     communityRelCount: Integer,   // 社区关系数
>     
>     // 配置信息
>     model: String,            // 使用的模型名称
>     gcsBucket: String,        // Google Cloud Storage 存储桶
>     gcsBucketFolder: String,  // GCS存储桶文件夹
>     gcsProjectId: String,     // GCS项目ID
>     awsAccessKeyId: String,   // AWS访问密钥ID
>     access_token: String      // 访问令牌
> })
> 
> // Document节点的关系
> (:Chunk)-[:PART_OF]->(:Document)      // Chunk属于Document的关系
> (:Chunk)-[:FIRST_CHUNK]->(:Document)  // 第一个Chunk与Document的关系
> ```
>
> ```cypher
> // Chunk节点结构
> (:Chunk {
>     // 基本信息
>     id: String,              // 块ID
>     text: String,            // 块的文本内容
>     position: Integer,       // 块在文档中的位置
>     length: Integer,         // 块的长度
>     fileName: String,        // 所属文件名
>     content_offset: Integer, // 内容偏移量
>     
>     // 页面信息（针对PDF等分页文档）
>     page_number: Integer,    // 页码（可选）
>     
>     // 时间信息（针对视频/音频文档）
>     start_time: String,      // 开始时间（可选）
>     end_time: String,        // 结束时间（可选）
>     
>     // 向量嵌入
>     embedding: List[Float]   // 文本的向量表示
> })
> 
> // Chunk节点的关系
> (:Chunk)-[:PART_OF]->(:Document)           // 属于某个文档
> (:Chunk)-[:FIRST_CHUNK]->(:Document)       // 作为文档的第一个块
> (:Chunk)-[:NEXT_CHUNK]->(:Chunk)           // 与下一个块的连接
> (:Chunk)-[:HAS_ENTITY]->(:Entity)          // 与实体的关系
> (:Chunk)-[:SIMILAR]->(:Chunk)              // 与相似块的关系
> ```
>
> ```cypher
> // Entity节点结构
> (:__Entity__ {
>     // 基本信息
>     id: String,              // 实体ID
>     description: String,     // 实体描述
>     
>     // 向量表示
>     embedding: List[Float],  // 实体的向量嵌入
>     
>     // 社区信息
>     communities: List[Int],  // 实体所属的社区ID列表
>     
>     // 动态标签
>     // Entity节点可以有多个标签，除了__Entity__基础标签外，
>     // 还可以有表示实体类型的具体标签，如Person、Organization等
> })
> 
> // Entity节点的关系
> (:Chunk)-[:HAS_ENTITY]->(:__Entity__)                    // 从Chunk到Entity的关系
> (:__Entity__)-[:IN_COMMUNITY]->(:__Community__)          // Entity属于某个社区
> (:__Entity__)-[relationship_type]->(:__Entity__)         // Entity之间的关系（可以有多种类型）
> 
> ```
>
> ```cypher
> // 1. 基础系统关系类型
> (:Chunk)-[:PART_OF]->(:Document)           // 块属于文档
> (:Chunk)-[:NEXT_CHUNK]->(:Chunk)           // 块之间的顺序关系
> (:Chunk)-[:FIRST_CHUNK]->(:Document)       // 第一个块与文档的关系
> (:Chunk)-[:HAS_ENTITY]->(:__Entity__)      // 块与实体的关系
> (:__Entity__)-[:IN_COMMUNITY]->(:__Community__)  // 实体属于社区
> (:__Community__)-[:PARENT_COMMUNITY]->(:__Community__)  // 社区的层级关系
> (:Chunk)-[:SIMILAR]->(:Chunk)              // 块之间的相似关系
> 
> // 2. 实体间的动态关系
> // 实体之间可以有多种类型的关系，这些关系是动态生成的
> // 例如：
> (:Person)-[:WORKS_AT]->(:Company)
> (:Person)-[:KNOWS]->(:Person)
> (:Company)-[:LOCATED_IN]->(:Location)
> // 等等...
> 
> // 3. 关系的属性
> // 每个关系都有以下基本属性：
> {
>     element_id: String,           // 关系的唯一标识符
>     type: String,                 // 关系类型
>     start_node_element_id: String,// 起始节点ID
>     end_node_element_id: String   // 终止节点ID
> }
> 
> // 4. 关系的特点
> // - 关系是有方向的（->）
> // - 关系类型使用大写字母和下划线命名
> // - 系统会自动排除某些内置关系类型：
> //   'PART_OF', 'NEXT_CHUNK', 'HAS_ENTITY', '_Bloom_Perspective_',
> //   'FIRST_CHUNK', 'SIMILAR', 'IN_COMMUNITY', 'PARENT_COMMUNITY'
> 
> // 5. 关系的创建和管理
> // - 关系在创建时需要确保起始和终止节点都存在
> // - 可以通过MERGE语句创建关系以避免重复
> // - 支持批量创建和管理关系
> ```



源码解析：

```python
def create_communities(uri, username, password, database, model=COMMUNITY_CREATION_DEFAULT_MODEL):
    try:
        # 1. 初始化图数据科学驱动
        gds = get_gds_driver(uri, username, password, database)
        # 2. 清除已有社区
        clear_communities(gds)
        # 3. 创建图投影
        graph_project = create_community_graph_projection(gds)
        # 4. 执行社区检测
        write_communities_sucess = write_communities(gds, graph_project)
        if write_communities_sucess:
            # 5. 创建社区属性
            create_community_properties(gds,model)
```



## 创建图投影create_community_graph_projection

```python
CREATE_COMMUNITY_GRAPH_PROJECTION = """
MATCH (source:{node_projection})-[]->(target:{node_projection})
WITH source, target, count(*) as weight
WITH gds.graph.project(
               '{project_name}',
               source,
               target,
               {{
               relationshipProperties: {{ weight: weight }}
               }},
               {{undirectedRelationshipTypes: ['*']}}
               ) AS g
RETURN
  g.graphName AS graph_name, g.nodeCount AS nodes, g.relationshipCount AS rels
"""
```

- 节点匹配

  ```cypher
  MATCH (source:{node_projection})-[]->(target:{node_projection})
  ```

  - `{node_projection} `是一个参数，根据代码它的值是 `!Chunk&!Document&!__Community__`，表示匹配除了`Chunk`、`Document`和`_Community_`之外的所有节点类型

  - `source` 和` target` 分别代表关系的起点和终点

  - `-[]->` 表示匹配任意类型的关系（没有指定关系类型）

- 权重计算

  ```cypher
  WITH source, target, count(*) as weight
  ```

  - 对于每一对节点（`source`和`target`），计算它们之间的关系数量

  - `count(*) `统计这两个节点之间所有的关系数量

  - 这个数量被赋值给 `weight` 变量，作为这对节点之间的边权重

- 图投影创建

  ```cypher
  WITH gds.graph.project(
      '{project_name}',  // 图投影的名称
      source,           // 源节点
      target,           // 目标节点
      {
          relationshipProperties: { weight: weight }  // 定义关系属性
      },
      {undirectedRelationshipTypes: ['*']}  // 图的配置
  ) AS g
  ```

  这部分使用Neo4j的图数据科学库（GDS）创建图投影：

  - `{project_name}`：投影图的名称，通常是"communities"

  - `source, target`：指定投影图的节点

  - `relationshipProperties`：定义关系属性

    ``weight: weight`：将前面计算的weight值作为关系的权重属性

  - `undirectedRelationshipTypes: ['*']`

    `'*'` 表示包含所有类型的关系

    `undirected `表示将有向图转换为无向图，这意味着 A->B 和 B->A 会被视为相同的边

    > - GDS会将指定的图数据从Neo4j数据库加载到内存中
    >
    > - 创建一个优化的内存数据结构来存储这个图
    >
    >   这个投影的实际结构可能是这样的结构
    >
    >   ```text
    >   投影图 "communities" {
    >       节点A ----weight:2----> 节点B
    >       节点B ----weight:1----> 节点C
    >       节点A ----weight:3----> 节点D
    >       ...
    >   }	
    >   ```
    >
    > - 这个内存中的图投影可以被后续的图算法使用，比如社区检测算法
    >
    > - 投影会一直存在于内存中，直到：
    >   - 显式调用删除（如上面代码中的gds.graph.drop）
    >   - Neo4j服务器重启
    >   - 内存不足时可能被自动清理

- 返回结果

  ```cypher
  RETURN
      g.graphName AS graph_name,  // 返回创建的图投影名称
      g.nodeCount AS nodes,       // 返回图中的节点数量
      g.relationshipCount AS rels // 返回图中的关系数量
  ```

  - `graph_name`: 返回创建的图投影的名称

    例如: 如果投影名称设置为"communities"，则可能返回 "communities"

  - `nodes`: 返回图投影中包含的节点总数

    例如: 如果图中有100个实体节点，则返回 100

  - `rels`: 返回图投影中包含的关系总数

    例如: 如果这些节点之间有150个连接关系，则返回 150



## 执行社区检测算法write_communities

```python
gds.leiden.write(
            { # 举例，使用图映射的返回
  						"graph_name": "communities",
  						"nodes": 100,
  						"rels": 150
						},
            writeProperty='communities',
            includeIntermediateCommunities=True,
            relationshipWeightProperty="weight",
            maxLevels=MAX_COMMUNITY_LEVELS,
            minCommunitySize=MIN_COMMUNITY_SIZE,
        )
```

- `writeProperty`: 指定写入社区标识的属性名

- `includeIntermediateCommunities`: 设为True表示保存中间层次的社区结构

- `relationshipWeightProperty`: 使用边的"weight"属性作为社区检测的权重

- `maxLevels`: 最大社区层次数为3,意味着可以发现3层嵌套的社区结构

- `minCommunitySize`: 最小社区大小为1,表示允许单个节点构成社区

使用Leiden算法检测社区，写入图数据库。

- 算法的结果会被写入到每个实体节点的属性中

- 属性名为 ·`communities`（由writeProperty=project_name指定）

- 每个节点的 `communities` 属性是一个数组，包含该节点在每一层的社区ID

**经过算法检测之后，并没有生成社区节点`_Communtity_`，而是给实体节点添加了社区标识**。标识也就是序号。

举个例子，经过这一步之后下面三个entity实体增加的社区属性：

```text
Node A: communities = [1, 4, 7]  // 三层社区的ID
Node B: communities = [1, 4, 7]  // 和A在同一社区
Node C: communities = [2, 5, 8]  // 不同社区
```



## 创建社区create_community_properties

```python
def create_community_properties(gds, model):
    commands = [
        (CREATE_COMMUNITY_CONSTRAINT, "created community constraint to the graph."),
        (CREATE_COMMUNITY_LEVELS, "Successfully created community levels."),
        (CREATE_COMMUNITY_RANKS, "Successfully created community ranks."),
        (CREATE_PARENT_COMMUNITY_RANKS, "Successfully created parent community ranks."),
        (CREATE_COMMUNITY_WEIGHTS, "Successfully created community weights."),
        (CREATE_PARENT_COMMUNITY_WEIGHTS, "Successfully created parent community weights."),
    ]
    try:
        for command, message in commands:
            gds.run_cypher(command)
            logging.info(message)

        create_community_summaries(gds, model)
        logging.info("Successfully created community summaries.")

        embedding_dimension = create_community_embeddings(gds)
        logging.info("Successfully created community embeddings.")

        create_vector_index(gds=gds,index_type=ENTITY_VECTOR_INDEX_NAME,embedding_dimension=embedding_dimension)
        logging.info("Successfully created Entity Vector Index.")

        create_vector_index(gds=gds,index_type=COMMUNITY_VECTOR_INDEX_NAME,embedding_dimension=embedding_dimension)
        logging.info("Successfully created community Vector Index.")

        create_fulltext_index(gds=gds,index_type=COMMUNITY_FULLTEXT_INDEX_NAME)
        logging.info("Successfully created community fulltext Index.")

    except Exception as e:
        logging.error(f"Error during community properties creation: {e}")
        raise
```

### 第一阶段：基础属性创建

```python
commands = [
    (CREATE_COMMUNITY_CONSTRAINT, "created community constraint to the graph."),
    (CREATE_COMMUNITY_LEVELS, "Successfully created community levels."),
    (CREATE_COMMUNITY_RANKS, "Successfully created community ranks."),
    (CREATE_PARENT_COMMUNITY_RANKS, "Successfully created parent community ranks."),
    (CREATE_COMMUNITY_WEIGHTS, "Successfully created community weights."),
    (CREATE_PARENT_COMMUNITY_WEIGHTS, "Successfully created parent community weights."),
]
```

- **CREATE_COMMUNITY_CONSTRAINT创建社区节点ID的唯一性约束**

  ```sql
  CREATE CONSTRAINT IF NOT EXISTS FOR (c:__Community__) REQUIRE c.id IS UNIQUE;
  ```

- **CREATE_COMMUNITY_LEVELS创建社区层级结构**

  ```sql
  MATCH (e:`__Entity__`)
  WHERE e.communities is NOT NULL
  UNWIND range(0, size(e.communities) - 1 , 1) AS index
  CALL {
    WITH e, index
    WITH e, index
    WHERE index = 0
    MERGE (c:`__Community__` {id: toString(index) + '-' + toString(e.communities[index])})
    ON CREATE SET c.level = index
    MERGE (e)-[:IN_COMMUNITY]->(c)
    RETURN count(*) AS count_0
  }
  CALL {
    WITH e, index
    WITH e, index
    WHERE index > 0
    MERGE (current:`__Community__` {id: toString(index) + '-' + toString(e.communities[index])})
    ON CREATE SET current.level = index
    MERGE (previous:`__Community__` {id: toString(index - 1) + '-' + toString(e.communities[index - 1])})
    ON CREATE SET previous.level = index - 1
    MERGE (previous)-[:PARENT_COMMUNITY]->(current)
    RETURN count(*) AS count_1
  }
  RETURN count(*)
  ```

  - **基础数据匹配**
  
    ```sql
      MATCH (e:`__Entity__`)
      WHERE e.communities is NOT NULL
      UNWIND range(0, size(e.communities) - 1 , 1) AS index
    ```
  
      - 匹配所有有communities属性的实体节点
      - 使用UNWIND展开一个范围数组，范围从0到communities数组长度减1
      - 这样可以遍历每个实体的每一层社区
  
  - **处理第0层社区**
  	```sql
    CALL {
      WITH e, index
      WHERE index = 0
      MERGE (c:`__Community__` {id: toString(index) + '-' + toString(e.communities[index])})
      ON CREATE SET c.level = index
      MERGE (e)-[:IN_COMMUNITY]->(c)
      RETURN count(*) AS count_0
    }
    ```
  
    - 处理最底层社区（level 0）
  
    - 创建社区节点，ID格式为"0-社区号"
  
    - 设置社区的level属性为0
  
    - 创建实体到社区的IN_COMMUNITY关系
  
    > 假设我们的实体数据是：
    >
    > ```text
    > Entity1: communities = [5, 2, 1]
    > Entity2: communities = [3, 2, 1]
    > Entity3: communities = [5, 2, 1]
    > ```
    >
    > 经过这一个子查询之后生成的数据是
    >
    > ```text
    > (Entity1)-[:IN_COMMUNITY]->(__Community__ {id: "0-5"})
    > (Entity2)-[:IN_COMMUNITY]->(__Community__ {id: "0-3"})
    > (Entity3)-[:IN_COMMUNITY]->(__Community__ {id: "0-5"})
    > ```
    
  - **处理其他层级社区**
  
    ```sql
    CALL {
      WITH e, index
      WHERE index > 0
      MERGE (current:`__Community__` {id: toString(index) + '-' + toString(e.communities[index])})
      ON CREATE SET current.level = index
      MERGE (previous:`__Community__` {id: toString(index - 1) + '-' + toString(e.communities[index - 1])})
      ON CREATE SET previous.level = index - 1
      MERGE (previous)-[:PARENT_COMMUNITY]->(current)
      RETURN count(*) AS count_1
    }
    ```
    
    - 处理更高层级的社区（level > 0）
    
    - 创建当前层级的社区节点
    
    - 创建前一层级的社区节点（如果不存在）
    
    - 建立前一层级社区到当前社区的PARENT_COMMUNITY关系
    
    假设有一个实体节点的communities = [3, 1, 4]，执行后的图结构将变为：
    
    ```text
    (Entity) -[:IN_COMMUNITY]-> (Community "0-3")
    (Community "0-3") -[:PARENT_COMMUNITY]-> (Community "1-1")
    (Community "1-1") -[:PARENT_COMMUNITY]-> (Community "2-4")
    ```
    
    每个Community节点的属性：
    
    ```json
    {
        "0-3": { "id": "0-3", "level": 0 },
        "1-1": { "id": "1-1", "level": 1 },
        "2-4": { "id": "2-4", "level": 2 }
    }
    ```
  
- **CREATE_COMMUNITY_RANKS计算每个社区文档数量排名**

  ```sql
  MATCH (c:__Community__)<-[:IN_COMMUNITY*]-(:!Chunk&!Document&!__Community__)<-[HAS_ENTITY]-(:Chunk)<-[]-(d:Document)
  WITH c, count(distinct d) AS rank
  SET c.community_rank = rank;
  ```

  让我们拆解这个复杂的匹配路径

  ```sql
  (c:__Community__)<-[:IN_COMMUNITY*]-(:!Chunk&!Document&!__Community__)<-[HAS_ENTITY]-(:Chunk)<-[]-(d:Document)
  ```

  `(c:__Community__)`

  - 起始点是社区节点

  - 标签为__Community__

  `<-[:IN_COMMUNITY*]-(:!Chunk&!Document&!__Community__)`

  - 匹配实体节点(通过排除法指定)

  - !Chunk&!Document&!__Community__表示不是这些标签的节点，即Entity节点

  - IN_COMMUNITY*表示可能有多跳的IN_COMMUNITY关系

  `<-[HAS_ENTITY]-(:Chunk)`

  - 匹配到Chunk节点

  - 通过HAS_ENTITY关系连接到实体

  `<-[]-(d:Document)`

  - 最终匹配到Document节点

  - 使用任意类型的关系连接到Chunk

  

  这一步是为了**计算每个社区影响或覆盖的文档范围**

- **CREATE_PARENT_COMMUNITY_RANKS 计算高层社区的文档数量**

  ```sql
  MATCH (c:__Community__)<-[:PARENT_COMMUNITY*]-(:__Community__)<-[:IN_COMMUNITY*]-(:!Chunk&!Document&!__Community__)<-[HAS_ENTITY]-(:Chunk)<-[]-(d:Document)
  WITH c, count(distinct d) AS rank
  SET c.community_rank = rank;
  ```

  这一步相比上一步变化的点在于`(c:__Community__)`变成了`(c:__Community__)<-[:PARENT_COMMUNITY*]-(:__Community__)`。

  ==为什么==

  - **只有level 0的社区通过:IN_COMMUNITY关系直接连接到实体**

  - **更高层级的社区只通过:PARENT_COMMUNITY关系相互连接**

  - 清晰的层级结构

  - 避免实体与多个层级社区的重复关联

  - 通过父子关系可以轻松追踪任意层级的关联实体

- **CREATE_COMMUNITY_WEIGHTS计算社区权重**

  ```sql
  MATCH (n:`__Community__`)<-[:IN_COMMUNITY]-()<-[:HAS_ENTITY]-(c)
  WITH n, count(distinct c) AS chunkCount
  SET n.weight = chunkCount
  ```

  也就是统计每个社区关联的实体数量。

- **CREATE_PARENT_COMMUNITY_WEIGHTS计算高层社区权重**

  ```sql
  MATCH (n:`__Community__`)<-[:PARENT_COMMUNITY*]-(:`__Community__`)<-[:IN_COMMUNITY]-()<-[:HAS_ENTITY]-(c)
  WITH n, count(distinct c) AS chunkCount
  SET n.weight = chunkCount
  ```

  高层社区的实体数量。

### 第二阶段：生成社区摘要

```python
def create_community_summaries(gds, model):
    try:
        # 第一阶段：处理基础社区
        community_info_list = gds.run_cypher(GET_COMMUNITY_INFO)
        community_chain = get_community_chain(model)
        
        # 并行处理基础社区摘要
        summaries = []
        with ThreadPoolExecutor() as executor:
            futures = [executor.submit(process_community_info, community, community_chain) 
                      for community in community_info_list.to_dict(orient="records")]
            
            for future in as_completed(futures):
                result = future.result()
                if result:
                    summaries.append(result)
                    
        # 存储基础社区摘要
        gds.run_cypher(STORE_COMMUNITY_SUMMARIES, params={"data": summaries})

        # 第二阶段：处理父级社区
        parent_community_info = gds.run_cypher(GET_PARENT_COMMUNITY_INFO)
        parent_community_chain = get_community_chain(model, is_parent=True)
        
        # 并行处理父级社区摘要
        parent_summaries = []
        with ThreadPoolExecutor() as executor:
            futures = [executor.submit(process_community_info, community, parent_community_chain, is_parent=True) 
                      for community in parent_community_info.to_dict(orient="records")]
            
            for future in as_completed(futures):
                result = future.result()
                if result:
                    parent_summaries.append(result)
                    
        # 存储父级社区摘要
        gds.run_cypher(STORE_COMMUNITY_SUMMARIES, params={"data": parent_summaries})

    except Exception as e:
        logging.error(f"Failed to create community summaries: {e}")
        raise
```

跟之前一样，处理0层社区和高层社区。对于每一层社区的操作，都是获取相关信息、交给大模型提取标题和摘要、保存进数据库。

那么我们来看看获取相关信息和大模型提示词的区别：

- 获取相关信息

  - 0层社区

    ```python
    GET_COMMUNITY_INFO = """
    MATCH (c:`__Community__`)<-[:IN_COMMUNITY]-(e)
    WHERE c.level = 0
    WITH c, collect(e) AS nodes
    WHERE size(nodes) > 1
    CALL apoc.path.subgraphAll(nodes[0], {
        whitelistNodes:nodes
    })
    YIELD relationships
    RETURN c.id AS communityId,
           [n in nodes | {id: n.id, description: n.description, type: [el in labels(n) WHERE el <> '__Entity__'][0]}] AS nodes,
           [r in relationships | {start: startNode(r).id, type: type(r), end: endNode(r).id}] AS rels
    """
    ```

    通过实体获取社区ID、节点信息、关系信息。

  - 高层社区

    ```python
    GET_PARENT_COMMUNITY_INFO = """
    MATCH (p:`__Community__`)<-[:PARENT_COMMUNITY*]-(c:`__Community__`)
    WHERE p.summary is null and c.summary is not null
    RETURN p.id as communityId, collect(c.summary) as texts
    """
    ```

    通过社区关联关系，获取子社区摘要。

- 大模型提示词

  - 0层社区

    ```python
    COMMUNITY_TEMPLATE = """
    Based on the provided nodes and relationships that belong to the same graph community,
    generate following output in exact format
    title: A concise title, no more than 4 words,
    summary: A natural language summary of the information
    {community_info}
    """
    ```

  - 高层社区

    ```python
    PARENT_COMMUNITY_TEMPLATE = """
    Based on the provided list of community summaries that belong to the same graph community, 
    generate following output in exact format
    title: A concise title, no more than 4 words,
    summary: A natural language summary of the information. Include all the necessary information as much as possible.
    {community_info}
    """
    ```

### 第三阶段：生成社区向量嵌入

```python
def create_community_embeddings(gds):
    try:
        embedding_model = os.getenv('EMBEDDING_MODEL')
        embeddings, dimension = load_embedding_model(embedding_model)
        logging.info(f"Embedding model '{embedding_model}' loaded successfully.")
        
        logging.info("Fetching community details.")
        rows = gds.run_cypher(GET_COMMUNITY_DETAILS)
        rows = rows[['communityId', 'text']].to_dict(orient='records')
        logging.info(f"Fetched {len(rows)} communities.")
        
        batch_size = 100
        for i in range(0, len(rows), batch_size):
            batch_rows = rows[i:i+batch_size]            
            for row in batch_rows:
                try:
                    row['embedding'] = embeddings.embed_query(row['text'])
                except Exception as e:
                    logging.error(f"Failed to embed text for community ID {row['communityId']}: {e}")
                    row['embedding'] = None
            
            try:
                logging.info("Writing embeddings to the database.")
                gds.run_cypher(WRITE_COMMUNITY_EMBEDDINGS, params={'rows': batch_rows})
                logging.info("Embeddings written successfully.")
            except Exception as e:
                logging.error(f"Failed to write embeddings to the database: {e}")
                continue
        return dimension
    except Exception as e:
        logging.error(f"An error occurred during the community embedding process: {e}")
```



### 第四阶段：创建社区相关索引

- 创建Entity向量索引

  ```python
  create_vector_index(gds=gds,index_type=ENTITY_VECTOR_INDEX_NAME,embedding_dimension=embedding_dimension)
  ```

- 创建社区向量索引

  ```python
  create_vector_index(gds=gds,index_type=COMMUNITY_VECTOR_INDEX_NAME,embedding_dimension=embedding_dimension)
  ```

- 创建社区摘要的全文索引

  ```python
  create_fulltext_index(gds=gds,index_type=COMMUNITY_FULLTEXT_INDEX_NAME)
  ```

  
