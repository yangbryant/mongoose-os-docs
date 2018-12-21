# 应用程序和库

Mongoose OS **应用程序** 是做特殊功能的固件.构建刷机到微处理器上. 例如, [blynk app](https://github.com/mongoose-os-apps/blynk) 是用[Blynk mobile app](https://blynk.cc)控制设备的固件.

另一个例子是[default app](https://github.com/mongoose-os-apps/default), 当你在 Web UI "设备控制"页面上点击 "刷机",或者输入 `mos flash <arch>` 命令, 开始刷机. 默认程序周期性闪烁LED灯, 能和MQTT服务器通信, 允许用户在设备文件系统上直接编辑 JavaScript 代码来扩展逻辑.

应用程序可以使用任意数量的库. **lib** 是可重用的库. 它不能直接构建到运行的固件中, 因为它只提供API, 实际上并没有用API. 应用程序在[mos.yml](https://github.com/mongoose-os-apps/empty/blob/master/mos.yml)文件的`libs:`部分添加库. `mos build` 命令生成调用库初始化方法的代码, 库按照参考的顺序初始化.

## 本地和远程构建

默认情况下, `mos build` 命令构建应用程序固件, 使用的所说的远程构建方式 - 打包应用程序的源码, 发送到 Mongoose OS 构建服务器. 这是默认的操作, 因为他不需要在本地安装[Docker](http://docker.com/).

但是, 如果本地安装了 Docker, 就可以本地构建. 添加额外的`--local` 参数(如下)来完成. 这种情况下, 一切都在本地完成. 对于自动构建的, 以及不希望源码离开本地的人来说, 这是更好的选择. 如下:

| Build type | Build command |
| ---------- | ------------- |
| Remote (default)    | <pre>mos build --platform PLATFORM</pre> |
| Local  (requires Docker)   | <pre>mos build --platform PLATFORM --local --verbose</pre> |

## mos.yml 文件格式参考

`mos.yml` 文件驱动 Mongoose 应用程序构建的方式. 下面是文件字段(键值)的详情说明. 库也有 `mos.yml` 文件, 唯一的区别是库有`type: lib`的键值对, 导致它不能被构建成固件. 因此, 以下内容适用于应用程序和库.

### author

作者, 字符串格式 `FirstName SecondName <Email>`, 例如:
```yaml
author: Joe Bloggs <joe@bloggs.net>
```

### build_vars

Makefile列表. ?构建应用程序所需的传递给特定芯片架构的Makefile?. 详情请参考下一节构建过程. 特定芯片架构的Makefile示例:[fw/platforms/esp32/Makefile.build](https://github.com/cesanta/mongoose-os/blob/master/fw/platforms/esp32/Makefile.build). 其他的架构在各自的路径下: `fw/platforms/*/Makefile.build`.

以下示例通过禁用掉电检测来改变 ESP32 SDK 配置:

```yaml
build_vars:
  ESP_IDF_SDKCONFIG_OPTS: "${build_vars.ESP_IDF_SDKCONFIG_OPTS} CONFIG_BROWNOUT_DET="
```

另一个是启动 DNS-SD 的 [dns-sd library](https://github.com/mongoose-os-libs/dns-sd/blob/master/mos.yml) 库:
```yaml
build_vars:
  MGOS_ENABLE_MDNS: 1
```

### binary_libs

`.a` 库或者库所在路径的列表. 路径名不要写尾部斜杠:

```yaml
binary_libs:
  - mylib/mylib.a
```

### cdefs

传递给编译器的其他的预处理标志, 例如:

```yaml
cdefs:
  FOO: BAR
```

在C/C++源码, 会转换成 `-DFOO=BAR` 编译选项.

### cflags, cxxflags

修改 C (`cflags`) 和 C++ (`cxxflags`) 的编译标志. 例如, 默认情况下, 警告被视为错误. 此设置在编译 C 代码时忽略警告:

```yaml
cflags:
  - "-Wno-error"
```

如果你想定义预处理变量, `cdefs` 更简单. 如下:

```yaml
cdefs:
  FOO: BAR
```

同样的写法:

```yaml
cflags:
  - "-DFOO=BAR"
cxxflags:
  - "-DFOO=BAR"
```

### config_schema

它可以为设备定义新的配置段, 也可以覆盖之前其他地方定义的配置. 例如, 下面的代码片段定义了新的段 `foo`, 覆盖了 `mqtt` 库中 `mqtt.server` 的默认值:

```yaml
config_schema:
  - ["foo", "o", {title: "my app settings"}]
  - ["foo.enable", "b", true, {title: "Enable foo"}]
  - ["mqtt.server", "1.2.3.4:1883"]
```

> 译者注: int = "i", bool = "b", double = "d", string = "s", object = "o", 参考于[Mongoose-OS源码](https://github.com/cesanta/mongoose-os/blob/master/fw/tools/gen_sys_config.py).

### description

字符串格式, 一行简短描述, 例如:

```yaml
description: Send BME280 temperature sensor readings via MQTT
```

### filesystem

包含要复制到设备文件系统的文件的文件或目录列表, 例如:

```yaml
filesystem:
  - fs
  - other_dir_with_files
  - foo/somepage.html
```

### includes

C/C++ 头文件路径列表. 路径名不要写尾部斜杠. 例如:

```yaml
includes:
  - my_stuff/include
```

### libs

库依赖项. 每个库都有 `origin`, 可以选择有 `name` 和 `version`. `origin` 是 Github 地址, 如 `https://github.com/mongoose-os-libs/aws` (注意: 必须是在仓库根目录下有 `mos.yml` 文件的仓库!).

`name` 用来生成调用库初始化方法的代码: 例如, 如果库名字是 `mylib`, 应该有名字为 `bool mgos_mylib_init(void)` 的方法. 同时, 对于本地构建, 名字还用来作为 `deps` 下的路径名: 那是 `mos` 克隆库源码的位置.

`version` 可以是 git 标签名, 分支名, 或者仓库的SHA值. 如果忽略, 默认是 `mos.yml` 文件里的 `libs_version`, `libs_version` 默认为 mos 工具版本. 因此, 例如, 如果 mos 工具版本是 1.21, 那么默认使用 tag 为 `1.21` 的库. 最新的 mos 会使用 `master` 分支.
例如:

```yaml
libs:
    # Use aws lib on the default version
  - origin: https://github.com/mongoose-os-libs/aws

    # Use aws lib on the version 1.20
  - origin: https://github.com/mongoose-os-libs/aws
    version: 1.20

    # Use the lib "mylib" located at https://github.com/bob/mylib-test1
  - origin: https://github.com/bob/mylib-test1
    name: mylib
```

### name

覆盖应用程序或者库名称. 默认情况下, 这个名字设置和路径名相同.

```yaml
name: my_cool_app
```

### sources

包含要C/C++源文件或文件目录的列表. 路径名不要写尾部斜杠:

```yaml
sources:
  - src
  - foo/bar.c
```


### tags

用于 Web UI 搜索的自由格式的字符串标签列表. 一些标签是预定义的, 它们把应用程序或者库划分到某一分类中. 预编译的标签有: `cloud` (云集成), `hardware` (硬件设备或API), `remote_management` (远程管理), `core` (核心功能). 例如:

```yaml
tags:
  - cloud
  - JavaScript
  - AWS
```

## 深入了解构建流程

当在应用程序路径下执行 `mos build [FLAGS]` 命令时, 执行了如下行为:

- `mos` 扫描了在 `mos.yml` 文件里的 `libs:` 段, 将所有库导入到 libs 路径下(`~/.mos/libs`, 可以通过 `--libs-dir ANOTHER_DIR` 标志覆盖)

- 每个库也都有 `mos.yml` 文件, 每个库也可能有 `libs:` 段 - 这样这个库可以依赖其他库. `mos` 以递归的方式导入所有依赖库.

- 当所有必需的库导入后, `mos` 在每个库执行 `git pull` 以便更新. 可以用 `--no-libs-update` 标志关闭更新.

- 此时, 所有必需的库导入并更新.

- `mos` 将应用程序的 `mos.yml` 文件和所有依赖库的 `mos.yml` 文件合并成一个文件. 合并文件的顺序是: 如果 `my-app` 依赖 `lib1` 库, 然后 `lib1` 库依赖 `lib2` 库, 那么 `result_yml = lib2/mos.yml + lib1/mos.yml + my-app/mos.yml`. 意味着, 应用程序的 `mos.yml` 有最高的优先级.

- 如果指定了 `--local --verbose --repo PATH/TO/MONGOOSE_OS_REPO` 标志, 那么 `mos` 调用 `docker.cesanta.com/ARCH-build` docker镜像开始本地构建. 这个镜像涵盖了用 [Mongoose OS 源码](https://github.com/cesanta/mongoose-os)封装指定芯片架构的原生 SDK. `mos` 工具调用指定平台的 `make -f fw/platforms/ARCH/Makefile.build` 文件. docker 调用的结果就是生成一个包含构建文件和 `build/fw.zip` 固件压缩文件的 `build/` 文件夹, 压缩文件用来使用 `mos flash` 命令刷设备固件.

- 如果没有指定 `--local` 标志, 则将源码和文件系统文件打包发送到 [Mongoose OS 云构建平台](http://mongoose.cloud), 执行上一步中所述的构建, 然后发回一个带有 `build/fw.zip` 和构建文件的 `build/` 文件夹.
- 在 `build/` 路径下生成如下构建文件:

```bash
build/fw.zip  - a built firmware
build/fs      - a filesystem directory that is put in the firmware
build/gen     - a generated header and source files
```

## 怎样创建新库

- 开发新库的最好的方式是作为应用程序开发的一部分. 在应用程序中, 执行本地构建来创建 `deps/` 路径. 这就是你应该放置新库的路径.
- 克隆新库框架的 `empty` 库到 `deps/mylib` 路径 (更改 `mylib` 为你想要的名字): `git clone https://github.com/mongoose-os-libs/empty deps/mylib`
- 在你的库里创建 `include/mgos_mylib.h` 和 `src/mgos_mylib.c` 文件:

  #### mgos_mylib.c:

```c
#include "mgos_mylib.h"

// NOTE: library init function must be called mgos_LIBNAME_init()
bool mgos_mylib_init(void) {
  return true;
}
```

  #### mgos_mylib.h:

```c
#include "mgos.h"
```

- 将在 `mgos_mylib.h` 添加指定库的API接口, 并在 `mgos_mylib.c` 中实现.
- 在应用程序的 `mos.yml` 文件中添加新库:

```yaml
libs:
  - name: mylib
```

- 点击构建按钮构建应用程序, 刷机按钮刷机.
- 编辑库源码文件 `mylib/src`,构建 `myapp`, 测试应用程序按预期运行.

## 如何移植Arduino库

- 按照上一节列出的步骤操作.
- 将 Arduino 库源码复制到 `mylib/src` 目录, 并将 .h 头文件复制到 `include/` 路径下.
- 对 C++ API 添加 C 的封装, 使它能够成为 JS FFI 的封装: 在 API 中使用简单的类型, 最多6个32位参数, 2个64位参数. 具体参考[https://github.com/cesanta/mjs#cc-interoperability](https://github.com/cesanta/mjs#cc-interoperability)
- 如果计划也支持 JavaScript, 用 FFI JS 封装器创建 `mjs/api_mylib.js` 文件.
- 构建 / 测试 `myapp` 正常运行.
- 请参阅示例库 [https://mongoose-os.com/docs/reference/api.html#hardware](https://mongoose-os.com/docs/reference/api.html#hardware)

## 贡献应用程序或库

如果你想在社区共享项目,遵循[Apache 2.0 license](https://en.wikipedia.org/wiki/Apache_License)协议发布, 请按照以下步骤:

- 按照上一节的说明构建应用程序, 刷新固件,测试成功.
- 修改 `mos.yml`, 设置 `author` 字段为 `Your Name <your@email.address>`.
- 确保有一个描述性质的 `README.md` 文件.
- 如果是库:
    - 如果你的库支持 JavaScript API, 请创建 `mjs_fs/api_<name>.js` 文件.
    - 如果是 Arduino 库的移植, 确保在 `mos.yml`文件里有 `arduino-compat` 库, 请参阅示例库 [arduino-adafruit-ssd1306 lib](https://github.com/mongoose-os-libs/arduino-adafruit-ssd1306/blob/master/mos.yml).
    - 请参阅获取参考 [https://github.com/mongoose-os-libs/blynk](https://github.com/mongoose-os-libs/blynk)
    - 考虑贡献你使用库的示例应用程序.
- 使用主题 `New contribution: ...` [在论坛里发起讨论](https://forum.mongoose-os.com/post/discussion/mongoose-iot),
	显示代码在 GitHub / Bitbucket / 其他的链接, 或者提交源代码的压缩文件.
