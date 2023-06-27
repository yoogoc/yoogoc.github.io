---
title: 向量数据库-qdrant-基本使用
tags:
- db
- rust
- ai
date: 2023-06-05 17:13:00

---

# 什么是向量数据库

向量数据库是一种专门用于存储和查询向量数据的数据库系统。它不同于传统的关系型数据库或文档数据库，它们主要关注存储和查询结构化或半结构化数据。向量数据库的主要目标是高效地处理和检索向量数据，这些数据可以是图像、音频、文本或其他形式的向量表示。

矢量数据库经过优化，可以有效地**storing**和**querying **高维矢量。通常使用特定的数据结构和索引技术，比如qdrant使用的**Hierarchical Navigable Small World (HNSW)(分层导航小世界?)**  -- 用于实现近似最近邻 和产品量化（Product Quantization）等。

向量数据库支持快速相似性和语义搜索，同时允许用户根据一些距离度量找到最接近给定查询向量的向量。

## Product Quantization

是一种用于高维向量索引和检索的技术，常用于向量数据库中。它的主要用途是将高维向量数据映射到低维码本（codebook）中，以便在有限的存储空间和计算资源下进行高效的近似搜索。

以下是Product Quantization的一些主要用途：

1. 高维向量索引：在大规模向量数据库中，传统的线性扫描方法在效率上往往受限。PQ能够将高维向量压缩为多个子向量，并将它们分别存储在不同的码本中。这样一来，搜索时只需计算查询向量与码本之间的距离，而不需要遍历整个数据库。这种索引方式可以大大加速向量搜索的速度。
2. 近似搜索：由于PQ进行了向量的近似编码，因此使用PQ进行搜索时，返回的结果往往是近似匹配的。在一些实际应用中，近似搜索已经足够满足需求，而且可以极大地降低计算复杂度。
3. 内存和存储优化：PQ可以将高维向量数据压缩为较低维度的码本，从而大大减少存储需求。这对于大规模向量数据集的存储和处理非常有帮助，特别是在资源受限的环境中。
4. 分布式计算：PQ可以方便地分割和并行处理向量数据，因为每个子向量和码本之间的计算是相互独立的。这种并行化的特性使得PQ适用于分布式计算环境，可以更高效地处理大规模数据。

总而言之，Product Quantization是一种能够提供高效近似搜索的向量编码方法，它在向量数据库中具有广泛的应用，包括高维向量索引、近似搜索、内存和存储优化以及分布式计算等领域。

## Scalar Quantization

一种向量数据编码方法，与Product Quantization（PQ）相对应。与PQ将向量分解为多个子向量并对其进行编码不同，SQ将整个向量视为一个标量值，并将其量化为离散的数值。Scalar Quantization在向量数据库中具有以下主要用途：

1. 空间压缩：SQ可以将高维向量数据压缩为较低维度的离散数值，从而减少存储空间的需求。这对于具有大量向量数据的数据库非常有用，可以显著减少存储成本。
2. 数据传输和通信：由于SQ将向量编码为离散数值，这种编码方式在数据传输和通信中非常有效。将向量数据量化为离散数值后，可以更容易地进行传输和解码，减少了传输的数据量和带宽需求。
3. 快速计算：由于SQ将向量量化为标量值，因此可以利用标量操作的快速计算特性。在一些计算密集型任务中，SQ可以提供高效的计算性能，减少了向量操作的复杂性和计算开销。
4. 索引和搜索：虽然SQ在索引和搜索方面相对于PQ有一些限制，但在某些应用场景下仍然具有一定的用途。使用SQ进行索引和搜索时，可以将向量编码为离散数值，并利用这些数值进行索引和匹配。这种方法可以在一定程度上提高搜索效率。

需要注意的是，SQ相对于PQ来说，损失了更多的向量信息，因为它将整个向量视为一个标量进行编码。因此，在需要精确匹配和高质量结果的应用中，PQ通常更常用。但在一些资源受限或对近似匹配要求较低的场景下，SQ可以作为一种简单且高效的向量编码方式。

## Hierarchical Navigable Small World（HNSW）算法

一种用于高维向量索引和搜索的算法。它是一种基于图的索引结构，旨在提供高效的近似最近邻搜索。

HNSW算法的主要思想是构建一个多层级的图结构，其中每个节点表示一个向量，并通过边连接到其他节点。这个图结构允许快速导航和搜索。

以下是HNSW算法的关键特点和用途：

1. 多层级结构：HNSW使用多层级的图结构来组织向量数据。每个节点在不同的层级上具有不同的连接，形成一个层级关系。这种结构有助于提高搜索效率和减少计算开销。
2. 近似搜索：HNSW是一种近似搜索算法，它通过维护一个"小世界"网络，使得相似的向量更有可能在相邻节点之间连接。这样一来，当进行近似搜索时，可以通过快速遍历"小世界"网络，跳过不相关的节点，以快速找到候选的近邻。
3. 动态更新：HNSW支持动态数据集的更新。当向量数据库需要插入新的向量时，HNSW可以有效地将其插入到图结构中，并保持索引的一致性和性能。
4. 高维向量索引：HNSW在处理高维向量索引时表现出良好的性能。传统的线性扫描方法在高维空间中往往效率低下，而HNSW通过构建多层级结构和近似搜索的方式，可以在高维空间中快速找到候选的最近邻。

HNSW算法在大规模高维向量数据集的索引和搜索中具有广泛的应用。它在许多实际场景中展示了优秀的性能，包括图像检索、文本搜索、推荐系统等领域。通过构建多层级的图结构和近似搜索的方式，HNSW能够提供高效的近似最近邻搜索能力，并适用于资源受限的环境。

# qdrant是什么

读作**quadrant**,是一个向量相似度搜索引擎，它提供了一个生产级的服务，并且有方便的api去存储、搜索和管理points（带有payload的向量）。Qdrant 专为扩展过滤支持而定制（其实我不明白这个定制是什么意思，后面阅读代码的时候要注意下）。它可用于各种神经网络或基于语义的匹配、分面搜索和其他应用程序。

# 基本使用

## 安装方式

与其他newsql一样，都是二进制安装或docker或k8s，更详细的说明建议看官网：

1. 二进制：`cargo build --release --bin qdrant`

2. docker：`docker run -p 6333:6333 -v $(pwd)/path/to/data:/qdrant/storage qdrant/qdrant`

3. k8s helm: 

   ```sh
   helm repo add qdrant https://qdrant.to/helm
   helm install qdrant qdrant/qdrant
   ```


### CRUD

对于向量数据库而言，只有两个维度，一个是集合的增删改查，一个是point的增删改查。

#### 集合

很常规的操作，直接看文档即可。

#### point

对于point的操作有很多，但也很清晰：

1. Get point：通过id获取单个point完整信息
2. Get points：通过指定ID检索多个point
3. Upsert points：对点执行插入+更新。如果给定 ID 的点已存在则覆盖。
4. Delete points：通过指定ID删除多个point
5. Update vectors：更新point指定的命名向量，保持未指定命名的向量完整。
6. Delete vectors：从指定ID删除命名向量。
7. Set payload：设置指定ID的point的payload
8. Overwrite payload：将point的全部payload替换为新的payload
9. Delete payload：从指定key删除point的payload
10. Clear payload：清空指定ID的point的payload
11. Scroll points：对符合给定过滤条件的point的分页列表
12. Search points：根据向量相似度和给定过滤条件检索符合条件的point的分页列表
13. Search batch points：批量根据向量相似度和给定过滤条件检索符合条件的point的分页列表
14. Search point groups：根据向量相似度和给定过滤条件检索符合条件的point，并根据给定的分组字段进行分组
15. Recommend points：寻找更接近存储的正例但同时更接近负例的点。
16. Recommend batch points：批量寻找更接近存储的正例但同时更接近负例的点。
17. Recommend point groups：寻找更接近存储的正例但同时更接近负例的点，并根据给定的分组字段进行分组
18. Count points：统计符合给定过滤条件的point的数量

### 文档

https://qdrant.github.io/qdrant/redoc/index.html

# 基本概念

### 距离指标（distance mertice）

距离指标可以用做向量相似度，指标的选择取决于向量的获取方式，特别是神经网络编码器（neural network encoder）训练的方法。

qdrant支持以下几种类型：

1. 点积（dot product）
2. 余弦（cosine）
3. 欧式（euclidean）

#### cosine的实现

为了提高搜索效率，将余弦的实现转化为了点积，为什么会提高效率呢：

qdrant的实现中，会对存入的向量预处理，然后搜索时使用点积做相似度：

```rust
pub fn cosine_preprocess(vector: &[VectorElementType]) -> Option<Vec<VectorElementType>> {
    let mut length: f32 = vector.iter().map(|x| x * x).sum();
    if length < f32::EPSILON {
        return None;
    }
    length = length.sqrt();
    Some(vector.iter().map(|x| x / length).collect())
}

// 这个是保底实现
pub fn dot_similarity(v1: &[VectorElementType], v2: &[VectorElementType]) -> ScoreType {
    v1.iter().zip(v2).map(|(a, b)| a * b).sum()
}
```

在标准实现里：向量余弦 = dot_product(A, B) / (norm(A) * norm(B))

举例：A = [1,2,3], B = [4,5,6]

cosine_similarity(A,B) = (1\*4+2\*5+3\*6) /(sqrt(1^2+2^2+3^2)\*sqrt(4^2+5^2+6^2)) = 32/(sqrt(14)\*sqrt(77))

而在qdrant中的流程分成了两步：

1. 向量保存时：
   A = [1/sqrt(1^2+2^2+3^2),2/sqrt(1^2+2^2+3^2),3/sqrt(1^2+2^2+3^2)] = [1/sqrt(14), 2/sqrt(14), 3/sqrt(14)]
   B= [4/sqrt(4^2+5^2+6^2),5/sqrt(4^2+5^2+6^2),6/sqrt(4^2+5^2+6^2)] = [4/sqrt(77), 5/sqrt(77), 6/sqrt(77)]
2. 点积计算：
   dot_product(A, B) = (1/sqrt(14))\*(4/sqrt(77))+(2/sqrt(14))\*(5/sqrt(77))+(3/sqrt(14))\*(6/sqrt(77))
                                   = (1\*4/(sqrt(14)\*sqrt(77)))+(2\*5/(sqrt(14)\*sqrt(77)))+(3\*6/(sqrt(14)\*sqrt(77)))
                                   = 32/(sqrt(14)\*sqrt(77))

可以看到，最终的结果是一样的，但是，在查询时减少了计算两个向量范数，从而提高性能。

而在euclidean和dot_product计算中，qdrant也是利用了现代先进cpu的能力，使用avx/sse/neon指令集加速运算过程：

```rust
impl Metric for CosineMetric {
      ...
      fn similarity(v1: &[VectorElementType], v2: &[VectorElementType]) -> ScoreType {
        #[cfg(target_arch = "x86_64")]
        {
            if is_x86_feature_detected!("avx")
                && is_x86_feature_detected!("fma")
                && v1.len() >= MIN_DIM_SIZE_AVX
            {
                return unsafe { dot_similarity_avx(v1, v2) };
            }
        }

        #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
        {
            if is_x86_feature_detected!("sse") && v1.len() >= MIN_DIM_SIZE_SIMD {
                return unsafe { dot_similarity_sse(v1, v2) };
            }
        }

        #[cfg(all(target_arch = "aarch64", target_feature = "neon"))]
        {
            if std::arch::is_aarch64_feature_detected!("neon") && v1.len() >= MIN_DIM_SIZE_SIMD {
                return unsafe { dot_similarity_neon(v1, v2) };
            }
        }

        dot_similarity(v1, v2)
    }
    ...
}
```



## collection

collection是一些point的集合，可以理解为关系数据库中的table，但是需要注意的是，集合中point的向量的维度是确定且一致的，不然向量数据库无法计算相似度。

除了指标和向量大小，每个集合可以设置独立的一些控制参数，用来做集合优化，索引构造，vacuum

#### 创建集合

创建集合的protobuf参数

```protobuf
message CreateCollection {
  string collection_name = 1; // 集合名
  reserved 2; // Deprecated 废弃
  reserved 3; // Deprecated 废弃
  optional HnswConfigDiff hnsw_config = 4; // 向量索引配置
  optional WalConfigDiff wal_config = 5; // wal配置
  optional OptimizersConfigDiff optimizers_config = 6; // 优化器配置
  optional uint32 shard_number = 7; // 集合的分片数, 单机默认为1, 否则等于节点数. 最小值为1
  optional bool on_disk_payload = 8; // 如果为true，point的payload不会在内存中
  optional uint64 timeout = 9; // 等待操作commit超时秒数，如果不指定则为默认值
  optional VectorsConfig vectors_config = 10; // 向量配置
  optional uint32 replication_factor = 11; // 每个网络分片试图维护的副本数量，默认为1
  optional uint32 write_consistency_factor = 12; // 我们应该应用多少个副本才能认为操作成功，默认为1
  optional string init_from_collection = 13; // 指定名字则从其他集合中复制数据
  optional QuantizationConfig quantization_config = 14; // 向量量化配置
}

// hnsw-向量索引配置
message HnswConfigDiff {
  optional uint64 m = 1; //索引图中每个节点的边数。值越大——搜索越准确，需要的空间就越大。
  optional uint64 ef_construct = 2; // 建立索引时要考虑的邻居数目。值越大——搜索越准确，构建索引所需的时间就越多。
  /*
  用于附加基于有效负载的索引的向量的最小大小(以千字节为单位)。 
	如果有效负载块小于' full_scan_threshold '，则不会使用额外的索引 
	在这种情况下，查询规划器应该首选全扫描搜索，并且不需要额外的索引。 
	注意:1 Kb = 1个大小为256的向量
  */
  optional uint64 full_scan_threshold = 3;
  optional uint64 max_indexing_threads = 4; // 用于后台索引构建的并行线程数。如果是0 -自动选择。
  optional bool on_disk = 5; // 在磁盘上存储HNSW索引。如果设置为false，索引将存储在RAM中。
  optional uint64 payload_m = 6; // 索引图中每个节点的附加负载感知链接数。如果没有设置，将使用常规的M参数。
}

// wal配置
message WalConfigDiff {
  optional uint64 wal_capacity_mb = 1; // 单个wal文件大小
  optional uint64 wal_segments_ahead = 2; // 需要提前创建的segment数量
}

// 优化器配置
message OptimizersConfigDiff {
  optional double deleted_threshold = 1; // 段优化所需的被删除向量的最小比例。
  optional uint64 vacuum_min_vector_number = 2; // 进行段优化所需向量的最小数量。
  // 优化器尝试保持的segment的目标数量。
  // 实际段的数量可能会因多个参数而变化：
  // - 已经存储的point数量。
  // - 当前写入每秒请求数（RPS）。
  // 建议选择默认的segment数作为搜索线程数量的因子，这样每个segment可以被一个线程均匀处理。
  optional uint64 default_segment_number = 3; 
  /*
  不要创建大于此大小（以千字节为单位）的segment。
  较大的segment可能需要不成比例长的索引时间，因此限制segment的大小是有意义的。
  	- 如果索引速度更重要，请降低此参数。
		- 如果搜索速度更重要，请增加此参数。
	注意：1KB = 256大小的一个向量。
	如果未设置，将根据可用的CPU数量自动选择。
  */
  optional uint64 max_segment_size = 4;
  /*
  每个segment在内存中存储的向量的最大大小（以千字节为单位）。
  超过此阈值的段将以只读的内存映射文件形式存储。
  默认情况下，禁用了内存映射存储。要启用它，请将此阈值设置为合理的值。
	要禁用内存映射存储，请将其设置为0。
	注意：1KB = 大小为256的一个向量。
  */
  optional uint64 memmap_threshold = 5;
  /*
  允许用于普通索引的向量的最大大小（以千字节为单位），超过此阈值将启用向量索引。
  默认值为20,000，基于 https://github.com/google-research/google-research/blob/master/scann/docs/algorithms.md。
  要禁用向量索引，请设置为0。
	注意：1kB = 大小为256的一个向量。
  */
  optional uint64 indexing_threshold = 6;
  /*
  强制刷新之间的时间间隔。
  */
  optional uint64 flush_interval_sec = 7;
  /*
  可用于优化的最大线程数。如果设置为0，则会使用NUM_CPU - 1。
  */
  optional uint64 max_optimization_threads = 8;
}

// 向量配置
message VectorsConfig {
	// 如果一个point存在多个vector，那么可以对每个vector有不同的配置
  oneof config {
    VectorParams params = 1;
    VectorParamsMap params_map = 2;
  }
}

message VectorParams {
  uint64 size = 1; // 向量大小
  Distance distance = 2; // 计算距离指标的方法：cosine, dot, euclidean
  optional HnswConfigDiff hnsw_config = 3; // 向量HNSW配置。如果省略，则将使用集合的配置。
  optional QuantizationConfig quantization_config = 4; // 向量量化配置。如果省略，则将使用集合的配置。
  optional bool on_disk = 5; // 如果为true， 从disk上提供向量， 如果为false，向量会加载到ram。
}

message VectorParamsMap {
  map<string, VectorParams> map = 1; // map中的key为向量字段名
}

// 向量量化配置
message QuantizationConfig {
  oneof quantization {
    ScalarQuantization scalar = 1;
    ProductQuantization product = 2;
  }
}

// 标量量化
message ScalarQuantization {
  QuantizationType type = 1; // 量化类型
  optional float quantile = 2; // 用于量化的位数。
  optional bool always_ram = 3; // 如果设置为true，则无论主存储配置如何，量化向量始终将存储在内存中。
}

// 产品量化
message ProductQuantization {
  CompressionRatio compression = 1; // 压缩比
  optional bool always_ram = 2; // 如果设置为true，则无论主存储配置如何，量化向量始终将存储在内存中。
}

enum QuantizationType {
  UnknownQuantization = 0;
  Int8 = 1;
}

enum CompressionRatio {
  x4 = 0;
  x8 = 1;
  x16 = 2;
  x32 = 3;
  x64 = 4;
}

```

需要注意的是，如果一个collection持续增长，则需开启`hnsw_config.on_disk`或`vectors_config.on_disk`，将索引存储在硬盘上。这两个配置的区别是：hnsw代表向量索引存至硬盘，vectors代表向量本身存至硬盘。

同时，开启`optimizers_config.memmap_threshold`，官网给出的设置memmap_threshold值的规则：

1. 如果您有一个平衡的使用场景 - 将memmap_threshold设置为与indexing_threshold相同（默认为20000）。在这种情况下，优化器不会进行任何额外的运行，并且将一次性优化所有阈值。 
2. 如果您有高写入负载和较低的RAM - 将memmap_threshold设置为比indexing_threshold更低，例如10000。在这种情况下，优化器将首先将段转换为内存映射存储，然后再进行索引。 

### 调优策略

计算机没有银弹，qdrant需要再低内存，快速搜索，高精度中做妥协

#### 低内存+快速搜索

将向量保留在磁盘上，同时尽量减少磁盘读取的次数。

向量量化是实现这一目标的一种方法。量化将向量转换为更紧凑的表示形式，可以存储在内存中并用于搜索。通过使用较小的向量，您可以在内存中缓存更多数据，减少磁盘读取的次数。

举例的配置：

```
PUT /collections/{collection_name}

{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    },
    "optimizers_config": {
        "memmap_threshold": 20000
    },
    "quantization_config": {
        "scalar": {
            "type": "int8",
            "always_ram": true
        }
    }
}
```

mmmap_threshold参数将确保向量存储在磁盘上，而always_ram参数将确保量化向量存储在内存中。

更极端的做法，您可以通过搜索参数禁用重新评分，这将进一步减少磁盘读取的次数，但可能会稍微降低精度。

```
POST /collections/{collection_name}/points/search

{
    "params": {
        "quantization": {
            "rescore": false
        }
    },
    "vector": [0.2, 0.1, 0.9, 0.7],
    "limit": 10
}
```

#### 高精度+低内存

如果您需要高精度，但没有足够的 RAM 来在内存中存储向量，您可以启用磁盘向量和 HNSW 索引。

```
PUT /collections/{collection_name}

{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    },
    "optimizers_config": {
        "memmap_threshold": 20000
    },
    "hnsw_config": {
        "on_disk": true
    }
}
```

在这种情况下，即使 RAM 有限，您也可以通过增加 HNSW 索引的 ef 和 m 参数来提高搜索精度。

```
...
"hnsw_config": {
    "m": 64,
    "ef_construct": 512,
    "on_disk": true
}
...
```

在这种情况下，磁盘 IOPS 是一个关键因素，它将决定执行搜索的速度。您可以使用 fio 来测量磁盘 IOPS。

#### 高精度+快速搜索

对于高速和高精度搜索，在 RAM 中保存尽可能多的数据至关重要。默认情况下，Qdrant 遵循此方法，但您可以根据需要进行调整。

通过应用带有重新评分的量化，可以实现高搜索速度和可调精度。

```
PUT /collections/{collection_name}

{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    },
    "optimizers_config": {
        "memmap_threshold": 20000
    },
    "quantization_config": {
        "scalar": {
            "type": "int8",
            "always_ram": true
        }
    }
}
```

您还可以使用一些搜索时参数来调整搜索精度和速度：

```
POST /collections/{collection_name}/points/search

{
    "params": {
        "hnsw_ef": 128,
        "exact": false
    },
    "vector": [0.2, 0.1, 0.9, 0.7],
    "limit": 3
}
```

- hnsw_ef - 控制搜索期间要访问的邻居数量。值越高，搜索越准确，但速度越慢。推荐范围为 32-512。
- exact - 如果设置为 true，将执行精确搜索，速度会更慢，但更准确。您可以使用它来比较具有不同 hnsw_ef 值的搜索结果与基本事实。

#### 延迟与吞吐量

衡量搜索速度的主要方法有两种：

- 请求延迟 - 从提交请求到收到响应的时间
- 吞吐量 - 系统每秒可以处理的请求数

这些方法并不相互排斥，但在某些情况下，最好针对其中一种方法进行优化。

为了最小化延迟，您可以设置Qdrant以在单个请求中使用尽可能多的核心。您可以通过将集合中的段数设置为系统中的核心数来实现此目的。在这种情况下，每个段将并行处理，最终结果将更快地获得。

```
PUT /collections/{collection_name}

{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    },
    "optimizers_config": {
        "default_segment_number": 16
    }
}
```

为了优先考虑吞吐量，您可以设置Qdrant以尽可能多地使用核心来并行处理多个请求。为此，您可以将Qdrant配置为使用最小数量的段，通常为2个。大型段受益于索引的大小和寻找最近邻所需的向量比较次数较少。但与此同时，构建索引需要更长的时间。

```

PUT /collections/{collection_name}

{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    },
    "optimizers_config": {
        "default_segment_number": 2
    }
}
```

