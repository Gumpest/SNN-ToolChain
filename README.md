# SNN TOOLChainV2  

## SNN2CG

change

单元模块化 文件架构

```python
├── main.py-------------------------主函数
├── SNN_components------------------SNN组件
│  ├── spike_dag.py
│  ├── spike_layers.py--------------选择使用ann||snn的计算方式
│  └── spike_tensor.py--------------数据装载，频率编码
├── CG_components-------------------计算图组件
│  └── network.py
├── HW_components-------------------hardware 硬件组件
│  ├── axon.py----------------------axon 类定义
│  ├── core.py----------------------Core 类定义，具体实现对axon||neuron的set，路由等功能
│  ├── coreset.py-------------------封装Core
│  └── neuron.py--------------------neuron 类定义，具体实现LIF脉冲累积发放的过程
├── HW_settings---------------------硬件参数宏定义
│  └── settings.py
├── function------------------------功能项，主模块
│  ├── config_generator.py----------根据映射在core上的dict，地址编码，生成配置帧
│  ├── parser.py--------------------DAG网络参数解析
│  └── transformer.py---------------算子映射与芯片部署
├── imgs----------------------------图例
│  ├── info_format_example.png
│  ├── layer_info.png
│  └── new_graph.png
└── all.py--------------------------模块整合为一个文件
```

-----------------------------------

transformer.py ------- **trans_conv2d**  引入block的概念

#### step1：重新排列特征图 （仅支持空洞卷积的膨胀系数=1）

```python
def reorder(self, conv2d_config):
```

#### step2：选择合理的输入块大小（block_channel, block_h, block_w）

```python
def select_block_size(self, input_size, output_size, groups, kernel_size, stride, padding, weight_bits):
```

- AXON_NUM: 轴突数 1024
- 断言：input_channel * kw * kh / groups <= axon_num  至少能放一组channel
- 遍历C 、H、W，分别计算在这个三个维度需要的块数，三者相乘，并确定输入块尺寸
- 根据输入块尺寸、卷积核，计算输出块尺寸

```python
return ((bchannel, bh, bw), (output_bchannel, output_bh, output_bw))
```

#### step3：将每一个块映射到core上

- 求得三次遍历分别的次数
- 分块遍历映射
  - `def trans_one_block_conv2d`
    - 尝试使用**bias**
    - 遍历core_id，将neuron（输出）映射到core上，output_pos = [real_c, real_h, real_w]
    - `def output2input_conv2d(self, output_pos, conv2d_config):`
      - 由输出位置计算输入位置
      - 由输出channel位置计算输入channel位置
    - 将axon（输入）映射到core上，ind = CoreSet.set_one_axon(core_id, core_axon_pos + j, j)
    - 根据weight设置crossbar[]值									

#### step4：结果存储至dict中，返回axon, neuron, use_time



