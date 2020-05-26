# Ceph 存储集群

## 配置

### 存储设备

OSD可以通过两种方式管理它们存储的数据。从Luminous 12.2.z版本开始，新的默认（推荐）后端是BlueStore。在Luminous之前，默认（也是唯一的选项）是FileStore。

#### OSD后端

**BLUESTORE**

BlueStore是专用于存储的后端，专门用于管理Ceph OSD工作负载的磁盘上的数据。它是由过去十年中使用FileStore支持和管理OSD的经验所激发的。 BlueStore的主要功能包括：

* 直接管理存储设备。 BlueStore使用原始块设备或分区。这避免了可能影响性能或增加复杂性的任何中间抽象层（例如XFS之类的本地文件系统）
* RocksDB的元数据管理。我们嵌入RocksDB的键/值数据库是为了管理内部元数据，例如从对象名称到磁盘上块位置的映射
* 完整的数据和元数据校验和。默认情况下，所有写入BlueStore的数据和元数据都受到一个或多个校验和的保护。未经验证，不会从磁盘读取任何数据或元数据或将其返回给用户
* 内联压缩。写入的数据在写入磁盘之前可以选择压缩
* 多设备元数据分层。 BlueStore允许将其内部日志（预写日志）写入单独的高速设备（例如SSD，NVMe或NVDIMM），以提高性能。如果有大量的更快的存储空间，内部元数据也可以存储在更快的设备
* 高效的写时复制。 RBD和CephFS快照依赖于在BlueStore中有效实现的写时复制克隆机制。这将为常规快照和擦除代码池（依赖克隆实现有效的两阶段提交）提供高效的IO

**FILESTORE**

FileStore是在Ceph中存储对象的传统方法。它依赖于标准文件系统（通常为XFS）以及键/值数据库（传统为LevelDB，现在为RocksDB）来获取某些元数据

FileStore经过了充分的测试并广泛用于生产中，但是由于其总体设计和对用于存储对象数据的传统文件系统的依赖，因此存在许多性能缺陷

尽管FileStore通常能够在大多数POSIX兼容文件系统（包括btrfs和ext4）上运行，但是我们仅建议使用XFS。 Btrfs和ext4都有已知的错误和不足，使用它们可能会导致数据丢失。默认情况下，所有Ceph设置工具都将使用XFS

## 配置ceph

当您启动Ceph服务时，初始化过程将激活一系列在后台运行的守护程序。 Ceph存储集群运行三种类型的守护程序

* Ceph Monitor \(`ceph-mon`\)
* Ceph Manager \(`ceph-mgr`\)
* Ceph OSD Daemon \(`ceph-osd`\)

支持Ceph文件系统的Ceph存储群集至少运行一台Ceph元数据服务器（ceph-mds）。支持Ceph对象存储的集群运行Ceph网关守护程序（radosgw）

每个守护程序都有一系列配置选项，每个配置选项都有一个默认值。您可以通过更改这些配置选项来调整系统的行为

### 选项名称

所有Ceph配置选项都有一个唯一的名称，该名称由用小写字母组成的单词和下划线（\_）字符组成。

在命令行上指定选项名称时，下划线（\_）或破折号（-）字符可以互换使用（例如--mon-host等效于--mon\_host）。

当选项名称出现在配置文件中时，也可以使用空格代替下划线或破折号。

### 配置资源

每个Ceph守护进程，进程和库都将从以下列出的几个来源中提取其配置。如果同时存在，则列表后面的源将覆盖列表前面的源

* 编译的默认值
* 监视器集群的集中式配置数据库
* 存储在本地主机上的配置文件
* 环境变量
* 命令行参数
* 管理员设置的运行时替代

Ceph进程在启动时要做的第一件事就是解析通过命令行，环境和本地配置文件提供的配置选项。然后，该过程将与监视器群集联系，以检索整个群集集中存储的配置。一旦可获得完整的配置视图，则将继续执行守护程序或进程。

#### 启动选项

由于某些配置选项会影响流程与监视器联系，进行身份验证和检索集群存储的配置的能力，因此可能需要将其本地存储在节点上并在本地配置文件中进行设置。这些选项包括：

* mon\_host，集群的监视器列表
* mon\_dns\_serv\_name（默认值：ceph-mon），DNS SRV记录的名称，该记录用于检查以通过DNS识别集群监视器
* mon\_data，osd\_data，mds\_data，mgr\_data和类似的选项，它们定义守护程序将数据存储在哪个本地目录中
* keyring, keyfiel and/or key，可用于指定用于与监视器进行身份验证的身份验证凭据。请注意，在大多数情况下，默认密钥环位置在上面指定的数据目录中。

在大多数情况下，这些默认值都是合适的，但mon\_host选项除外，该选项标识群集的监视器的地址。使用DNS标识监视器时，可以完全避免使用本地ceph配置文件

#### 跳过监视器配置

可以通过--no-mon-config选项来传递任何进程，以跳过从集群监视器检索配置的步骤。在完全通过配置文件管理配置或监视器群集当前处于关闭状态但需要完成一些维护活动的情况下，这很有用

## 配置选项

任何给定的进程或守护程序的每个配置选项都有一个值。但是，选项的值可能在不同的守护程序类型之间变化，甚至是同一类型的守护程序也可能不同。存储在监视器配置数据库或本地配置文件中的Ceph选项分为几部分，以指示它们适用于哪些守护程序或客户端。

这些选项包括：

global

**Description**   global下设置会影响Ceph存储集群中的所有守护程序和客户端

**Example**  log\_file = /var/log/ceph/$cluster-$type.$id.log

mon

**Description**   mon下的设置会影响Ceph存储群集中的所有ceph-mon守护程序，并覆盖global中的相同设置

**Example**  mon\_cluster\_log\_to\_syslog = true

mgr

**Description**  mgr部分中的设置会影响Ceph Storage Cluster中的所有ceph-mgr守护进程，并覆盖global中的相同设置

**Example** mgr\_stats\_period = 10

osd

**Description**  osd下的设置会影响Ceph Storage Cluster中的所有ceph-osd守护进程，并覆盖global中的相同设置。

**Example**  osd\_op\_queue = wpq

mds

**Description**  mds部分中的设置会影响Ceph存储集群中的所有ceph-mds守护进程，并覆盖global中的相同设置

**Example**  mds\_cache\_memory\_limit = 10G

client

**Description**  client下的设置会影响所有Ceph客户端（例如已安装的Ceph文件系统，已安装的Ceph块设备等）以及Rados Gateway（RGW）守护程序

**Example**  objecter\_inflight\_ops = 512

部分还可以指定单个守护程序或客户端名称。例如，mon.foo，osd.123和client.smith都是有效的节名称   

在同一源（即，在同一配置文件中）的global，mon和mon.foo中都指定了同一选项，则将使用mon.foo值

如果在同一部分中指定了相同配置选项的多个值，则以最后一个值为准

请注意，本地配置文件中的值始终优先于监视器配置数据库中的值，而不管它们出现在哪个部分中

## 元变量

元变量极大地简化了Ceph存储集群的配置。在配置值中设置了元变量后，Ceph会在使用配置值时将元变量扩展为具体值。 Ceph元变量类似于Bash shell中的变量扩展

Ceph支持以下元变量：

$cluster

**Description**  扩展为Ceph存储群集名称。在同一硬件上运行多个Ceph存储集群时很有用

**Example**  /etc/ceph/$cluster.keyring

**Default**  ceph

$type

**Description**   扩展为守护程序或进程类型（例如mds，osd或mon）

**Example**   /var/lib/ceph/$type

$id

**Description**  扩展为守护程序或客户端标识符。对于osd.0，它将为0；否则为0。对于mds.a，它将是a

**Example** /var/lib/ceph/$type/$cluster-$id

$host

Description  扩展为运行进程的主机名。

$name

**Description**  扩展为$ type.$ id

**Example**  /var/run/ceph/$cluster-$name.asok

$pid

**Description**  扩展为守护进程pid。

**Example** /var/run/ceph/$cluster-$name-$pid.asok

## 配置文件

启动时，Ceph进程在以下位置搜索配置文件：

1. $CEPH\_CONF \(i.e._,_ the path following the $CEPH\_CONF environment variable\)
2. -c path/path \(i.e., the -c command line argument\)
3. /etc/ceph/$cluster.conf
4. ~/.ceph/$cluster.conf
5. ./$cluster.conf \(i.e., in the current working directory\)
6. On FreeBSD systems only, /usr/local/etc/ceph/$cluster.conf

其中$cluster是群集的名称（默认ceph）

Ceph配置文件使用ini样式的语法。您可以在注释之前添加井号（＃）或分号（;）。例如：

```text
# <--A number (#) sign precedes a comment.
; A comment may be anything.
# Comments always follow a semi-colon (;) or a pound (#) on each line.
# The end of the line terminates a comment.
# We recommend that you provide comments in your configuration file(s).
```

### 配置文件选项名

配置文件分为几部分。每个部分都必须以有效的配置部分名称（请参见上面的配置部分）开头，并用方括号括起来。例如

```text
[global]
debug ms = 0

[osd]
debug ms = 1

[osd.1]
debug ms = 10

[osd.2]
debug ms = 10
```

### 配置文件选项值

配置选项的值是一个字符串。如果太长而无法容纳在一行中，则可以在行尾添加一个反斜杠（\）作为行继续标记，因此该选项的值将是当前行中=之后的字符串与该字符串的组合在下一行：

```text
[global]
foo = long long ago\
long ago
```

在上面的示例中，"foo"的值为"long long age long ago"

通常，选项值以换行符或注释结尾，例如

```text
[global]
obscure one = difficult to explain # I will try harder in next release
simpler one = nothing to explain
```

在上面的示例中，“obscure one”的值将“difficult to explain”；而“simpler one ”的价值就是“nothing to explain”

如果选项值包含空格，并且我们想使其明确，则可以使用单引号或双引号将其引起来，例如

```text
line = "to be, or not to be"
```

某些字符不允许直接出现在选项值中。它们是=，＃、;和\[。如果需要，我们需要逃避它们，例如

```text
[global]
secret = "i love \# and \["
```

