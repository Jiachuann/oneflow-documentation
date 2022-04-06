# OneEmbedding
## Embedding
在推荐系统当中，常使用embedding方法，以下是在oneflow中进行embedding操作：
<!-- 我们经常会遇到离散特征，如userid、itemid。对于离散特征，我们一般的做法是将其转换为one-hot，
但对于itemid这种离散特征，转换成one-hot之后维度非常高，这种情况下，我们的通常做法就是将其转换为embedding。 -->
```python
import numpy as np
import oneflow as flow
indices = flow.tensor([[1, 2, 4, 5], [4, 3, 2, 9]], dtype=flow.int)
embed = flow.nn.Embedding(10, 3)
y = embed(indices)
```
## OneEmbedding
OneEmbedding是一个大规模Embedding的oneflow方案，可以解决大规模深度推荐系统的问题，它有以下特点：

（1）灵活的分层存储，支持将 Embedding table 放置在 GPU显存、CPU内存 、或者 SSD 上面，使用高速设备作为低速设备的缓存，实现速度与容量的兼顾。

（2）支持动态插入新特征ID。

## QuickRun 使用MultiTableEmbedding存储多个Embedding Table
### 第一步：导入相应包，设置对应配置
利用table_size_array定义词表大小。。。
```python
import oneflow as flow
import numpy as np
import oneflow.nn as nn
table_size_array = [39884407,39043,17289]
vocab_size = sum(table_size_array)
num_tables = len(table_size_array)
embedding_size = 128    
scales = np.sqrt(1 / np.array(table_size_array))
```
### 第二步：构造词表
使用make_table构造词表--flow.one_embedding.make_table(initializer)

根据table_size_array大小选择不同的初始化方式，可选的初始化类型有uniform和normal，分别通过make_uniform_initializer(low, high) 和make_normal_initializer(mean, std) 构造
```python
tables = [
    flow.one_embedding.make_table(
        low.one_embedding.make_uniform_initializer(low=-scale, height=scale)
    )
    for scale in scales
]
```
### 第三步：设置存储配置
store_options是OneEmbedding中词表的存储配置选项：
```python
store_options = flow.one_embedding.make_cached_ssd_store_options(
    cache_budget_mb=8142,persistent_path="/your_path_to_ssd", capacity=vocab_size,size_factor=1,   			
			physical_block_size=512
)
```

OneEmbedding提供了三种预置参数配置，分别是纯GPU存储配置，GPU CPU存储配置和GPU SSD存储配置，参数选项：

cache_budget_mb：每个GPU中作为词表高速缓存的显存大小，单位为MB；

persistent_path: Embedding词表持久化存储的路径，支持配置一个路径或一个列表：

    配置一个路径，代表分布式并行中各rank路径的根目录，会在该路径下创建num_rank个路径，名称格式为"rank_id-num_rank";

    配置一个列表，列表中每项代表分布式并行中每个rank，要求列表长度和num_rank一致;

    对于GPU SSD存储配置，训练过程中会对存储的词表进行频繁的数据读写，因此persistent_path 设置路径的文件随机读写速度对整体性能影响关键，因此最好使用高性能的SSD，如果用普通的磁盘，会对性能有很大影响;

capacity: Embedding词表总容量：

    对于纯GPU存储配置，词表全部存储在GPU显存上；

    对于GPU CPU存储配置，词表全部存储在CPU内存上，均会提前分配内存，因此对于纯GPU存储配置和GPU CPU存储配置必须设置capacity；

    对于GPU SSD存储配置，词表全部存储在SSD中，不需要提前分配内存，因此capacity是可选项，不是必须配置，但是如果配置了，有机会得到更好性能，因此如果事先知道规模，推荐配置capacity

size_factor： 词表存储大小和embedding_dim的比例

    若优化器为SGD，当momentum参数为0时，则只需要保存一份模型，size_factor设为1。当momentum参数大于0时，则不仅需要保存模型，还需要保存momentum状态，因此size_factor设为2；

    若优化器为Adam，则不仅需要保存模型，还需要保存m和v状态，因此size_factor设为3；
physical_block_size：Embedding词表持久化存储中使用的physical 
block size，physical block size应为磁盘扇区大小，一般为 512，physical_block_size 默认值是512，一般不用配置，若底层硬件设备的磁盘扇区大小是4096，则需要设置为4096。


### 第四步：实例化Embedding
使用MultiTableEmbedding实例化Embedding，其中name是Embedding词表的名称，embedding_dim是特征embedding的维度，dtype和key_type分别是embedding和特征id的数据类型
```python
embedding = flow.one_embedding.MultiTableEmbedding(
    name="my_embedding",
    embedding_dim=embedding_size,
    dtype=flow.float,
    key_type=flow.int64,
    tables=tables,
    store_options=store_options,
)
embedding.to("cuda")
mlp = flow.nn.FusedMLP(
    in_features=embedding_size * num_tables,
    hidden_features=[512, 256, 128],
    out_features=1,
    skip_final_activation=True,
)
mlp.to("cuda")
```
### 第五步：构造graph，进行训练
```python
class TrainGraph(flow.nn.Graph):
    def __init__(self,):
        super().__init__()
        self.embedding_lookup = embedding
        self.mlp = mlp
        self.add_optimizer(
            flow.optim.SGD(self.embedding_lookup.parameters(), lr=0.1, momentum=0.0)
        )
        self.add_optimizer(
            flow.optim.SGD(self.mlp.parameters(), lr=0.1, momentum=0.0)
        )
    def build(self, ids):
        embedding = self.embedding_lookup(ids)
        loss = self.mlp(flow.reshape(embedding, (-1, num_tables * embedding_size)))
        loss = loss.sum()
        loss.backward()
        return loss
ids = np.random.randint(0, 1000, (100, num_tables), dtype=np.int64)
ids_tensor = flow.tensor(ids, requires_grad=False).to("cuda")
graph = TrainGraph()
loss = graph(ids_tensor)
print(loss)
```
然后过渡到更细致更高级的功能。。。。

## 高阶 DLRM    
### OneEmbedding在DLRM任务上的应用

### 分布式扩展