# Build libwebrtc.a in Zorro

## Project Directory

```
zorro_android_sdk/src/
|- base/
|- build/
|- build_overrides/ 
|- buildtools/
|- examples/
|- out/
|- patches/
|- testing/
|- third_party/
|  |- ... # 和webrtc源码的third_party目录一样，而且把webrtc也放进来
|  |- webrtc/ # webrtc 源码修改
|  |  |- ... # webrtc源码目录
|  |  |- zorro/ # 一些增加的源码文件
|  |  |- BUILD.gn
|  |  |- webrtc.gni
|- tools/
|- BUILD.gn
```

## Build

因为只需要得到一个 `libwebrtc.a`， 所以 build target 只需要选定

```gn
# file: webrtc/BUILD.gn

rtc_static_library("webrtc") {...}
```

> build target 是GN工具的概念，GN将需要 build 的内容划分为一个个target然后在  `gn gen`的过程中补充每个target的信息。

只需要在`zorro_android_sdk/src/`下创建一个`BUILD.gn`，指向上述目标即可。

```
# 根目录 BUILD.gn 示例

group("default"){
    deps = []
    deps += [
        "//third_party/webrtc:webrtc",
    ]
}
```

当然，`webrtc/BUILD.gn`的内容也要改动，使得default target可以包含此目标

```
rtc_static_library("webrtc") {
    # Only the root target and the test should depend on this.
    visibility = [
      "//:default", # 新增，刚才在根目录创建的default target
      ".:default_",
      ":webrtc_lib_link_test",
    ]

    ...}
```
## Error in Source 

`third_party/webrtc/zorro/av_stream_decoder/av_stream_decoder.h`

加上头文件
```cpp
#include <condition_variable>
```


`third_party/webrtc/modules/audio_device/audio_device_impl.cc`

去掉头文件
```cpp
// #include "zorro/call_manager_interface_private.h"
```

## Command Lines

```shell
gn gen third_party/webrtc/out/m92 --args='is_debug=false is_component_build=false is_clang=true rtc_include_tests=false rtc_use_h264=true  rtc_enable_protobuf=false use_rtti=true use_custom_libcxx=false treat_warnings_as_errors=false use_ozone=true rtc_build_examples=false' && ninja -C third_party/webrtc/out/m92
```