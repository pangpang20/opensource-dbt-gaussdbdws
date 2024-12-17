# 在华为云部署项目

## Demo介绍

开发任务中对Demo的要求如下：

1.通过本地 GaussDB 作为数据源，通过 dbt-core 将数据转换成 GaussDB(DWS)适用的数据，通过 GaussDB(DWS) 的API上传数据;

2.GaussDB(DWS) 接收到转换后的数据，利用 GaussDB(DWS) 高效的数据加载、转换和查询性能等优势，生成目标数据;

3.最后，通过RESTful API操作数据库供前端使用。

经过分析，采用Jaffle Shop来实现Demo。

### Demo业务场景介绍

Jaffle Shop 假设了一家在线零售商的业务模型，模拟了客户购买行为的数据流转。其数据来源是CSV文件，包含 订单、客户和支付 等原始数据。这些数据经过 dbt-gaussdbdws 加载到GaussDB，在GaussDB中进行初步转换后，生成一系列干净、规范化的数据表。再同步到GaussDB(DWS),经过 dbt-gaussdbdws 的清洗，得到结果表（如汇总表或维度表），为分析师提供支持。

Jaffle Shop 是理解现代数据建模和转换的最佳实践的完美起点。

### Demo部署架构

下图是Demo的部署架构，基于CCE容器化部署：

![部署架构](../assets/2-0.png)

- 终端层

支持不同的类型终端访问Restful API接口

- 网关层

通过ELB进行负载均衡代理

- 中间层

在CCE中部署3个无状态服务：

jaffle-shop-gaussdb: 负责加载原始数据到GaussDB，并且在GaussDB中初加工数据

jaffle-shop-dws: 负责对GaussDB(DWS)中间表进行汇总加工，得到结果表

resource-service-python: 负责对GaussDB(DWS)结果表提供RESTful API接口

- 数据库层

GaussDB加工原始业务数据，主要负责初步清洗

GaussDB(DWS)加工汇总数据，得到结果表



### Demo数据流图

下图从整体逻辑上介绍了每个过程的数据处理：

![逻辑架构](../assets/2-1.png)


下图主要介绍在GaussDB中每个表的加工逻辑：

![数据流图](../assets/2-2.png)

下图主要介绍在GaussDB(DWS)中每个表的加工逻辑：

![数据流图](../assets/2-3.png)


### Demo开源地址

Demo中涉及的开源项目地址如下：

- [jaffle-shop-gaussdb](https://github.com/pangpang20/jaffle-shop-gaussdb.git) : 负责加载原始数据到GaussDB，并且在GaussDB中初加工数据
- [jaffle-shop-dws](https://github.com/pangpang20/jaffle-shop-dws.git) : 负责对GaussDB(DWS)中间表进行汇总加工，得到结果表
- [resource-service-python](https://github.com/pangpang20/resource-service-python.git) : 负责对GaussDB(DWS)结果表提供RESTful API接口

## 步骤一：云资源购买与配置

Demo部署主要依赖的云资源如下：

- VPC: 虚拟私有云，实现隔离
- ECS：制作镜像
- ELB：负载均衡和代理
- GaussDB：数据库加工原始数据
- GaussDB(DWS)：数据库加工汇总数据
- CCE：容器化部署
- SWR: 存放容器镜像
- NAT网关: 容器中访问公网

### 购买ECS并克隆项目

ECS配置如下：

- 购买数量：1台
- 规格：鲲鹏通用计算增强型 |  kc1.2xlarge.4 | 8vCPUs | 32GiB
- 镜像: Huawei Cloud EulerOS 2.0 标准版 64位 ARM版
- 登录凭证：密码
- 系统盘: 通用型SSD, 40GiB

具体购买操作可参考 [快速购买和使用Linux ECS](https://support.huaweicloud.com/qs-ecs/ecs_01_0103.html)

购买后，待服务器启动，登录服务器按下面的操作继续。

#### 安装Docker

参考 [基于EulerOS部署Docker](https://bbs.huaweicloud.com/blogs/441849)

#### 安装Git

参考 [基于EulerOS配置GitHub](https://bbs.huaweicloud.com/blogs/441850)

#### 克隆项目并且制作镜像

```bash
# 创建工作目录
mkdir mydemo
cd mydemo/

# 克隆 jaffle-shop-gaussdb
git clone https://github.com/pangpang20/jaffle-shop-gaussdb.git

# 制作镜像 jaffle-shop-gaussdb:1.0.0 (注意：需要一段时间，根据实际网络而定)
cd jaffle-shop-gaussdb
docker build -t jaffle-shop-gaussdb:1.0.0  .

# 返回工作目录，克隆 jaffle-shop-dws
cd ..
git clone https://github.com/pangpang20/jaffle-shop-dws.git

# 制作镜像 jaffle-shop-dws:1.0.0
cd jaffle-shop-dws
docker build -t jaffle-shop-dws:1.0.0  .


# 返回工作目录，克隆 resource-service-python
cd ..
git clone https://github.com/pangpang20/resource-service-python.git

# 制作镜像 resource-service-python:1.0.0
cd resource-service-python
docker build -t resource-service-python:1.0.0  .


# 查看镜像(3个镜像)
docker images

```

#### 上传镜像到SWR

进入`容器镜像服务 SWR`，点击 `组织管理` ,点击 `创建组织`

点击`总览`，点击右上角 `登录指令`，复制到ECS中执行

![登录指令](../assets/2-4.png)

上传镜像


```bash
sudo docker tag {镜像名称}:{版本名称} swr.cn-south-1.myhuaweicloud.com/{组织名称}/{镜像名称}:{版本名称}

sudo docker push swr.cn-south-1.myhuaweicloud.com/{组织名称}/{镜像名称}:{版本名称}

```

上传后在`我的镜像`中可以查看

![我的镜像](../assets/2-5.png)

### 购买GaussDB并创建用户和数据库


GaussDB配置如下：

- 数据库引擎: GaussDB
- 数据库引擎版本:V2.0-8.201
- 内核引擎版本:505.2.0
- 实例类型: 集中式版
- 部署形态: 1主2备
- 性能规格: 通用型（1:4）| 4 vCPUs | 16 GB
- 存储空间: 40 GB
- 数据库端口: 默认端口8000

具体购买操作可参考 [购买并通过界面化工具DAS连接GaussDB实例（推荐）](https://support.huaweicloud.com/qs-gaussdb/gaussdb_01_622.html)

购买后，待数据库实例启动，登录DAS创建数据库和创建用户。

> **🔔 注意：**
> 这里需要记录EIP或内网IP，数据库名，用户名，密码，后面需要。


### 购买GaussDB(DWS)并创建用户和数据库

GaussDB(DWS)配置如下：

- 版本选择：存算一体
- 部署类型: 集群
- 节点数量：3
- 存储容量：20G/节点
- 集群版本: 9.1.0.211


具体购买操作可参考 [快速创建GaussDB(DWS)集群并导入数据进行查询](https://support.huaweicloud.com/qs-dws/index.html)

购买后，待数据库实例启动，可参考[创建GaussDB(DWS)数据库和用户](https://support.huaweicloud.com/mgtg-dws/dws_01_01113.html)

> **🔔 注意：**
> 这里需要记录EIP或内网IP，数据库名，用户名，密码，后面需要。


### 购买CCE和节点


CCE配置如下：

- 集群类型CCE: Turbo
- 容器网络模型云原生网络: 2.0
- 集群版本: v1.30
- 集群规模: 50 节点
- 集群 master 实例数: 3实例（高可用）

CCE集群创建后，创建节点，节点配置如下：

- 节点类型：弹性云服务器-虚拟机
- 节点规格：鲲鹏通用计算增强型 | kc2.2xlarge.4 | 8 vCPUs | 32 GiB
- 容器引擎：Docker
- 操作系统：Huawei Cloud EulerOS2.0
- 登录方式：密码
- 磁盘：默认
- 节点数量：3

具体购买操作可参考 [在CCE集群中部署NGINX无状态工作负载](https://support.huaweicloud.com/qs-cce/cce_qs_0003.html)


### 购买ELB

ELB配置如下：

- 实例类型：独享型
- 实例规格：弹性规格，应用型+网络型
- 所属VPC：和CCE在同一个VPC
- 弹性公网IP带宽：10 Mbit/s

具体购买操作可参考 [实现单个Web应用的负载均衡](https://support.huaweicloud.com/qs-elb/zh-cn_topic_0052569751.html)

## 步骤二：部署Demo

在CCE中添加3个无状态的工作负载，具体如下：

### jaffle-shop-gaussdb

#### 创建并配置工作负载

参考下图配置基本信息：

![](../assets/3-1.png)

参考下图配置容器基本信息：

![](../assets/3-2.png)

参考下图配置容器环境变量，使用[GaussDB的信息](#gaussdb)：

![](../assets/3-3.png)

工作负载运行成功后如下：

![](../assets/3-4.png)


#### 进入容器执行数据处理

参考下图登录容器：

![](../assets/3-5.png)

![](../assets/3-6.png)

在容器中，测试GaussDB连接，使用下面的命令：

```bash
dbt debug

```
![](../assets/3-7.png)

测试数据在seeds目录：

```bash
ls -ltr seeds/

```
![](../assets/3-8.png)

执行dbt的数据导入，使用下面的命令：

```bash
dbt seed

```
![](../assets/3-9.png)

此时，可以在GaussDB中查看到数据已经导入：

```sql
ANALYZE jaffle_shop.raw_customers;
ANALYZE jaffle_shop.raw_items;
ANALYZE jaffle_shop.raw_orders;
ANALYZE jaffle_shop.raw_products;
ANALYZE jaffle_shop.raw_stores;
ANALYZE jaffle_shop.raw_supplies;

SELECT
    relname AS table_name,
    reltuples::BIGINT AS table_row_count
FROM
    pg_class c
JOIN
    pg_namespace n ON c.relnamespace = n.oid
WHERE
    n.nspname = 'jaffle_shop'
    AND c.relkind = 'r'  -- 只查询普通表
ORDER BY
    table_name;

```

![](../assets/3-10.png)

继续执行 dbt run 来运行项目,将raw开头的表经过数据清洗转换后加载到stg开头的表。

```bash
# 运行项目
dbt run

# 或者可以开启Debug模式，在运行过程中打印生成的每个 SQL 查询语句
dbt run -d --print
```

![](../assets/3-11.png)

在GaussDB中查看数据：

```sql
ANALYZE jaffle_shop.stg_customers;
ANALYZE jaffle_shop.stg_order_items;
ANALYZE jaffle_shop.stg_orders;
ANALYZE jaffle_shop.stg_products;
ANALYZE jaffle_shop.stg_locations;
ANALYZE jaffle_shop.stg_supplies;

SELECT
    relname AS table_name,
    reltuples::BIGINT AS table_row_count
FROM
    pg_class c
JOIN
    pg_namespace n ON c.relnamespace = n.oid
WHERE
    n.nspname = 'jaffle_shop'
    AND c.relkind = 'r'  -- 只查询普通表
    AND c.relname like 'stg%' -- 只查询stg开头的表
ORDER BY
    table_name;
```

![](../assets/3-12.png)

把GaussDB的数据通过GaussDB(DWS)的API导入到GaussDB(DWS)，按下面代码执行

```bash
# 复制配置文件
cp sample_trans_settings.yml trans_settings.yml

# 按说明修改参数
vi trans_settings.yml

# 执行数据迁移
source .venv/bin/activate
python3 datatrans.py
```
![](../assets/3-13.png)

此时，可以在GaussDB(DWS)中查看到数据：

SQL语句同GaussDB一样。

![](../assets/3-14.png)


### jaffle-shop-dws

#### 创建并配置工作负载

参考下图配置基本信息：

![](../assets/4-1.png)

参考下图配置容器基本信息：

![](../assets/4-2.png)

参考下图配置容器环境变量，使用[GaussDB(DWS)的信息](#gaussdbdws)：

![](../assets/4-3.png)

工作负载运行成功后如下：

![](../assets/4-4.png)


#### 进入容器执行数据处理

参考下图登录容器：

![](../assets/4-5.png)

在容器中，测试GaussDB(DWS)连接，使用下面的命令：

```bash
source .venv/bin/activate
dbt debug

```

![](../assets/4-6.png)


执行 `dbt run` 来运行项目,将stg开头的表经过数据清洗转换后加载结果表。

```bash
dbt run

```

![](../assets/4-7.png)

此时，可以在GaussDB(DWS)中查看到数据：

```sql
ANALYZE jaffle_shop.customers;
ANALYZE jaffle_shop.order_items;
ANALYZE jaffle_shop.orders;
ANALYZE jaffle_shop.products;
ANALYZE jaffle_shop.locations;
ANALYZE jaffle_shop.supplies;
ANALYZE jaffle_shop.metricflow_time_spine;

SELECT
    relname AS table_name,
    reltuples::BIGINT AS table_row_count
FROM
    pg_class c
JOIN
    pg_namespace n ON c.relnamespace = n.oid
WHERE
    n.nspname = 'jaffle_shop'
    AND c.relkind = 'r'
    AND c.relname not like 'stg%'
ORDER BY
    table_name;


```

![](../assets/4-8.png)


### resource-service-python

#### 创建并配置工作负载

参考下图配置基本信息：

![](../assets/5-1.png)

参考下图配置容器基本信息：

![](../assets/5-2.png)

参考下图配置容器环境变量，使用[GaussDB(DWS)的信息](#gaussdbdws)：

![](../assets/5-3.png)

参考下图配置服务：

![](../assets/5-4.png)


工作负载运行成功后如下：

![](../assets/5-5.png)


#### 访问API接口

参考下图获取访问地址：

![](../assets/5-6.png)

访问API（当然，DEMO没有做权限等控制）

![](../assets/5-7.png)

获取API数据

![](../assets/5-8.png)
