# 基础架构配置


Mongoose OS 使用结构化多层配置方式.
它由两部分组成: 定义配置的编译态部分和使用配置的运行态部分.

### 编译时生成

- 任何需要配置设置的代码片段都能定义一个描述该代码片段配置参数的 .yaml 文件.
- 用户代码可以在它自己的 .yaml 文件中定义它自己的一组配置参数.
- 所有 yaml 文件会在固件编译时合并生成一个文件 `sys_config_defaults.json`.
- 用户定义的 YAML 文件合并在文件的最后, 因此会覆盖系统 .yaml 文件中的指定的默认设置.
- 生成的 `sys_config_defaults.json` 文件声明了固件的所有可配置设置.
- 对应的头文件和源文件也会生成. 头文件包含了从 `sys_config_defaults.json` 文件转换得到的结构体, 以及获取和设置各种配置值的API.

![](images/config1.png)

### 运行时初始化

- `conf0.json` - 默认配置. 这是一份 `sys_config_defaults.json` 的副本. 最先加载, 必须在文件系统中存在. 其他所有的层都是可选的.
- `conf1.json` - `conf8.json` - 这些配置一层接一层的加载, 每个连续的配置层都会覆盖之前的(?由前一个层提供的 `conf_acl` 允许它?), 这些被用来覆盖?供应商配置?.
- `conf9.json` 是用户配置文件. 最后生效, 应用于所有其他层之上.
  `mos config-set` 和 `save_cfg()` API 方法修改 `conf9.json`.

![](images/config2.png)

因此有如下规则:

- 如果需要定义自己的配置参数，请在 `mos.yml` 文件中添加 `config_schema` 字段, 详细请参考 [quick start guide](/docs/quickstart/develop-in-c.md#create-custom-cofiguration-section)
- 如果要覆盖某些系统默认设置, 例如默认的 UART 速度, 也可以使用 `config_schema` 来覆盖, 请参考示例 [example](https://github.com/mongoose-os-apps/default/blob/c4e2acbb5fec8d151b0d74fa12f9f1791f08edeb/mos.yml#L23-L25) 
- 如果您想在每个固件上放置一些独一无二的信息, 例如唯一的ID, 并且可选择的保护数据不被进一步修改,  用 `conf1.json` - `conf8.json` 中的任意一层, 例如: `conf5.json`.
- `conf9.json` 不应该包含在固件内, 否则它将在OTA时覆盖用户的设置.
  
所以, 固件配置由一组 YAML 描述文件定义, 固件构建时这些文件转换成不透明的C的结构体 `mgos_sys_config` 和 公开的函数, 获取方法如 `mgos_sys_config_get_....()`, 设置方法如 `mgos_sys_config_set_....(value)`. C代码通过调用这些函数获取配置参数. 字段可以是 integer, boolean 或 string. 生成获取和保存全局配置的C方法.

举例怎样获取配置参数:

```c
  printf("My device ID is: %d\n", mgos_sys_config_get_device_id());  // Get config param
```

举例怎样设置配置参数和保存配置:

```c
  mgos_sys_config_set_debug_level(2); // Set numeric value
  mgos_sys_config_set_device_password("big secret"); // Set string value
  char *err = NULL;
  save_cfg(&mgos_sys_config, &err); /* Writes conf9.json */
  printf("Saving configuration: %s\n", err ? err : "no error");
  free(err);
```

这种生成机制不仅提供了好用的C API,还保证了如果C代码访问了某些参数, 它一定在描述文件中, 同时意味着它会在固件中. 这可以防止当配置重构或改变时常见的问题, 但C代码保持不变.

Mongoose OS 配置是可扩展的, 例如, 可以添加自己的配置参数, 这些参数可能是简单的, 也可能是复杂的(嵌套的).

在运行时, 配置由文件系统上的多个文件支持. 它有多个层: 默认值(0), ?供应商覆盖?(1-8), 和用户设置(9). 供应商层可以锁定用户层的部分配置, 允许某些字段更改. 例如: 最终用户可能改变 WiFi 设置, 但不能更改云端地址.

## 编译时生成深入研究

配置是由 Mongoose OS 代码库中的多个 YAML 文件定义的.每个 Mongoose OS 模块, 例如芯片加密模块, 都能在配置中定义它自己的部分. 例如:

- [mgos_sys_config.yaml](https://github.com/cesanta/mongoose-os/blob/master/fw/src/mgos_sys_config.yaml) is core module, defines debug settings, etc
- [mgos_atca_config.yaml](https://github.com/cesanta/mongoose-os/blob/master/fw/src/mgos_atca_config.yaml) is a crypto chip support module
- [mgos_mqtt_config.yaml](https://github.com/cesanta/mongoose-os/blob/master/fw/src/mgos_mqtt_config.yaml) has default MQTT server settings

此外, 你还可以到 [fw/src](https://github.com/cesanta/mongoose-os/tree/master/fw/src) 目录中查看.

如概述中所述, 你能在配置中定义自己的部分, 或者覆盖已存在的默认值. 设置方式是在 `mos.yml` 里设置 `config schema` 描述.例如:

```yaml
config_schema:
  - ["hello", "o", {"title": "Hello app settings"}]
  - ["hello.who", "s", "world", {"title": "Who to say hello to"}]
```

上面的代码来自 [fw/examples/c_hello](https://github.com/cesanta/mongoose-os/tree/master/fw/examples/c_hello/mos.yml).

固件构建完成后, 所有的 YAML 文件都合并为一个. 用户指定的 YAML 文件在最后一个, 因此它能覆盖其他配置.然后, 合并后的 YAML 文件转换成两个C文件 `mgos_config.h` 和 `mgos_config.c`. 固件构建完成后, 你能在 `YOUR_FIRMWARE_DIR/build/gen/` 路径下找到生成的文件.

这里有一个转换示例, 来自 [fw/examples/c_hello](https://github.com/cesanta/mongoose-os/tree/master/fw/examples/c_hello).
我们定义配置到 `src/conf_schema.yaml`:

```yaml
[
  ["hello", "o", {"title": "Hello app settings"}],
  ["hello.who", "s", "world", {"title": "Who to say hello to"}]
]
```

它会转换成如下的 getter 和 setter:

```c
const char *mgos_sys_config_get_hello_who(void);
void        mgos_sys_config_set_hello_who(const char *val);
```

然后,  在 [src/main.c](https://github.com/cesanta/mongoose-os/tree/master/fw/examples/c_hello/src/main.c) 的C固件代码, 通过如下方式获取自定义的配置值:

```c
  printf("Hello, %s!\n", mgos_sys_config_get_hello_who());
```

数字用 integer 表示, boolean 也是. 字符串使用堆分配内存.

**重要信息**: 空字符串指向 `NULL` 指针, 请注意.

实际上, 当前所有的子结构体都是公共的, 能用 getter 的方式访问到; 因此, 头文件包含结构体定义和 getter 方法:

```c
struct mgos_config_hello {
  char *who;
};

const struct mgos_config_hello *mgos_sys_config_get_hello(void);
```

将整个结构体作为参数的通用函数是很有用的. 在将来, 虽然可以选择公开某些特定的结构体, 但默认情况下所有结构体都是私有的.

## 运行态 - 工厂层, 供应商层, 用户层

设备配置保存在文件系统的如下几个文件中:

- `conf0.json` - 工厂默认层
- `conf1.json` to `conf8.json` - 供应商层
- `conf9.json` - 用户层

当 Mongoose OS 启动时, 它以完全相同的顺序读取这些文件, 合并成一个, 从 flash 到 memory 初始化C配置结构体. 因此, 在启动时, 结构体 `mgos_config` 按以下顺序进行初始化:

- 首先, 结构体置零.
- 第二, `conf0.json` 的默认值生效.
- 第三, _供应商层_ 1 到 8 一个接一个按顺序加载.
- 最后, _用户配置文件_ `conf9.json` 最后一步生效.

这个结果就是全局结构体 `mgos_config` 的状态. 每一步(层)都能覆盖一些,全部,或者不覆盖任何值. 默认值必须加载, 如果启动时文件不存在则会出错. 但是, 供应商和用户层是可选的.

注意, 默认情况下不存在供应商配置层. 它是为了便于后续设置的配置: 可以通过上传单一文件(例如通过 HTTP POST 上传到 /upload)来定制设备, 而不是执行完整的刷机. 供应商配置不能被"出厂重置"复位, 不管是通过GPIO方式还是Web方式.


## 字段访问控制

在配置中的某些设置可能是敏感的, 供应商会为用户提供修改设置的方法, 限制某些字段或者?用户更改字段的定义?.

为了方便起见, 配置系统包含有 **字段控制列表** (ACL) 管理的字段访问控制.

- ACL 是以逗号分隔的列表, 在启动加载配置文件时应用于完整的字段名称.
- ACL 按顺序匹配, 搜索到匹配后终止.
- ACL 是一种模式, `*` 用作通配符.
- ACL 以 `+` 或 `-` 开头, 如果符合匹配, 表示是否允许和拒绝修改字段. `+` 是隐含的, 但为了清楚被使用.
- ACL的默认值是 `*`, 允许更改任何字段.

ACL 包含在配置本身中 - 它是顶级字段 `conf_acl` . 稍微注意的是, 在加载过程中, _前一层_ 的设置会生效: 当加载用户设置时, 查询供应商设置的 `conf_acl` 值; 当加载供应商设置时, 默认使用的 `conf_acl` 值被使用.

例如, 为了限制用户只能修改 WiFi 和 debug level 设置, 在 `conf{1-8}.json` 中 应该设置 `"conf_acl": "wifi.*,debug.level"`.

否定条目允许默认允许的行为: 
`"conf_acl": "-debug.*,*"` 允许更改除了 `debug` 及其以下配置之外的所有字段.


## 重置为出厂默认值

如果配置了 `debug.factory_reset_gpio`, 在启动过程中保持指定的 pin 拉低, 消除用户设置(`conf9.json`).
注意, 如果存在供应商设置, 则不会被重置.
