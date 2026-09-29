# EVKphone — EVK5 事件相机安卓采集 App

在安卓手机上**免 root** 直接采集 Prophesee EVK5(以及 EVK3/EVK4 系列)事件相机数据的 App:

- 实时事件预览(红 = ON / 蓝 = OFF 事件,指数衰减)
- **三模态同步录制**:事件流(.raw)+ 手机内置 IMU(_imu.csv,硬件时间戳,字段兼容
  PC 端 record_event_imu_session.py)+ 手机后摄 1080p30 H.264(.mp4),同一 stem +
  _metadata.json 记录全部时间锚点(unix/elapsedRealtime/事件钟首末时间戳),录制到
  `/sdcard/evk_data/`(文件管理器/MTP 直接可见)
- 录制标准 **EVT3 RAW** 文件(可用 Metavision / OpenEB 工具直接回放)
- 事件率统计、ERC 事件率控制(USB2 手机限流必备)
- **坏点屏蔽(Digital Event Mask)**:在传感器数字管线内屏蔽坏点/热像素,被屏蔽像素的
  事件**根本不进 USB**——raw、预览、统计同时干净(PC 端实测坏点可占 0.4%~99% 事件量,
  本机 3 个已知坏点约吃掉 3M ev/s);坐标可增删、持久化保存、录制时写入 metadata
- **RAW 文件回放**:解析 `/sdcard/evk_data` 里的 .raw(EVT2/EVT3),流畅 1x 实时播放
  (事件时间切片 + 实时节拍),支持暂停/重播/进度拖动(RAW 索引 seek);
  预览背景为不透明纯黑(事件缓冲 alpha 初始化为 255,否则无事件区域会透出
  界面灰底,看起来像"雾")

## 技术路线

```
Kotlin UI (Compose)
   │  UsbManager → 权限 → UsbDeviceConnection.getFileDescriptor()
   ▼
libevkcam.so (JNI 桥, 本项目 native/evk-jni)
   │  setenv("EVKPHONE_USB_FD", fd)
   ▼
OpenEB HAL (libmetavision_hal / hal_discovery)          ← third_party/openeb(打了少量补丁)
   │  dlopen 插件
   ▼
libhal_plugin_prophesee.so + libmetavision_psee_hw_layer.so
   │  Treuzell 协议 (USB bulk, libusb)
   ▼
libusb1.0.so (libusb_wrap_sys_device 包装 java 传入的 fd)  ← third_party/libusb-master
   ▼
EVK5 硬件
```

要点:OpenEB 官方公开仓库不带安卓预编译依赖,本项目自建 libusb 并注入自定义
`ENV_CMAKE`;相机发现层改为从环境变量读取 app 打开的 USB fd,跳过需要 root 的
`/dev/bus/usb` 枚举。

## 目录结构

```
F:\EVKphone\
├─ app\                 Android 工程 (Kotlin + Compose)
├─ native\
│  ├─ evk-jni\          JNI 桥源码 (evk_camera_jni.cpp)
│  ├─ libusb-android\   libusb 交叉编译配置
│  └─ android\          android_env.cmake (OpenEB 的安卓环境注入)
├─ third_party\
│  ├─ openeb\           OpenEB 5.2.0 (已打补丁, 见下)
│  └─ libusb-master\    libusb 1.0.30
├─ scripts\
│  ├─ build_libusb.cmd  编译 libusb1.0.so
│  ├─ build_openeb.cmd  编译 OpenEB 最小库集 (5 个 .so)
│  ├─ build_jni.cmd     编译 libevkcam.so
│  ├─ package_libs.cmd  拷贝 .so 到 app jniLibs
│  └─ build_apk.cmd     Gradle 打 debug APK
├─ tools\gradle-8.9\    本地 Gradle
└─ build-android\       所有构建产物
```

## 构建(全流程)

前置:Android SDK + NDK 27 + CMake 3.22.1(本机已装于默认位置)。

```
scripts\build_libusb.cmd
scripts\build_openeb.cmd
scripts\build_jni.cmd
scripts\package_libs.cmd
scripts\build_apk.cmd
```

产物:`app\build\outputs\apk\debug\app-debug.apk`

## 对 OpenEB 打的补丁清单

| 文件 | 改动 |
|---|---|
| `CMakeLists.txt` (根) | ANDROID 时跳过 Boost/OpenCV/utils;`ENV_CMAKE` 传入时总是 include;`add_android_app.cmake` 缺失时跳过 |
| `cmake/Modules/FindLibUSB.cmake` | 已有 `libusb-1.0` imported target 时直接返回 |
| `hal_psee_plugins/src/boards/treuzell/tz_camera_discovery.cpp` | **核心补丁**:读取环境变量 `EVKPHONE_USB_FD`,用 `libusb_wrap_sys_device` 包装该 fd 并直接构建板卡命令,跳过 libusb 枚举;fd 变化时自动重包装 |
| `.../tz_libusb_board_command.{h,cpp}` | 构造函数增加 `adopted_handle` 参数;注入路径下放宽 VID/PID 匹配(接受任意 vendor-class 接口,仍校验 Treuzell 端点结构) |
| `.../utils/psee_libusb.{h,cpp}` | `LibUSBDevice` 增加"收养外部句柄"构造(不 open / 不 close fd);`LibUSBContext` 增加收养构造(配合 `libusb_init_context`) |
| `hal_psee_plugins/src/plugin/psee_universal.cpp` | **安卓实况相机修复 ①**:官方在 `__ANDROID__` 下不注册 Treuzell 相机发现(走不了 root 枚举);改为无条件注册 `TzCameraDiscovery`(04b4:00f4 等 USB ID),Fx3 发现保持仅桌面 |
| `.../treuzell/tz_camera_discovery.cpp` | **安卓实况相机修复 ②**:fd 注入分支移到 `LibUSBContext` 构造**之前**——原实现先构造 context,而安卓 SELinux 禁止枚举 `/dev/bus/usb`,`libusb_init` 直接抛 `LIBUSB_ERROR_IO`,注入路径永远执行不到;注入 context 用 `libusb_init_context(LIBUSB_OPTION_NO_DEVICE_DISCOVERY)` 创建(libusb 官方的 Android fd 注入配套模式);并带 logcat 直连诊断日志(tag evkcam) |
| `hal/.../i_event_decoder.h` + `hal/cpp/src/facilities/evk_template_instantiations.cpp` | **安卓跨库 RTTI 修复 ①**:`extern template` + 强实例化,消除 `I_EventDecoder<EventCD>` 模板类 vtable/typeinfo 在多个 .so 里的弱拷贝——否则插件以 `RTLD_LOCAL` dlopen 时 `Device::get_facility` 的 `dynamic_cast` 跨库失败(返回 null / 崩溃) |
| `hal/.../i_geometry.h` + 同上 .cpp | **安卓跨库 RTTI 修复 ②**:给全纯虚的 `I_Geometry` 补外置析构作为 key function,令其 typeinfo 在 libmetavision_hal 中强符号导出(同理修复文件设备 `geometry: 0x0` 问题) |

> RTTI 修复的背景:安卓上以 `RTLD_LOCAL` dlopen 插件时,无 key function 的类(纯虚接口、
> 模板类)的 typeinfo 会在各 .so 中重复存在且互不相等,导致 `dynamic_cast` 跨库返回
> null。有 key function 的类(如 `I_EventsStream`)天然不受影响。

## 使用

1. 安装 APK,OTG 线连接 EVK5;
2. App 里点「连接 VID xxxx」授权 USB 权限;
3. 自动:发现相机 → 打开 → 起流;
4. USB2 手机(如真我 GT8)保持 ERC 开启,按场景调 2M–100M ev/s;
   USB3 手机可关 ERC 跑满。

### 三模态录制(events + IMU + RGB)

1. 连接相机起流后,在「采集控制」里选择录制模态:IMU / 相机(相机首次使用需授权,
   后摄 1080p30 H.264 无声;IMU 默认开启);开启相机模态后 UI 会出现「RGB 预览」
   卡片实时显示后摄画面(录制中同样可见,便于取景);
2. 点「开始录制 (事件+IMU+相机)」,在 `/sdcard/evk_data/` 下同一 stem 生成 4 个文件:
   - `rec_*.raw` — 事件流(标准 EVT3);
   - `rec_*_imu.csv` — 手机内置 IMU,列名/顺序与 PC 端 record_event_imu_session.py
     完全一致(额外多一列 `sensor` 标注该行由哪个传感器事件触发:gyro/acc/rv);
     时间戳为 SensorEvent 硬件采样时刻(elapsedRealtimeNanos 时基);
   - `rec_*.mp4` — CameraX Recorder 输出;
   - `rec_*_metadata.json` — `time_base.unix_minus_monotonic_ns`、
     `event_camera.first/last_event_ts_us`(事件钟首末,来自解码回调精确记录)、
     record_start/end 等锚点;三模态对齐方式:IMU/视频直接在手机钟上,事件钟按
     record_start + first_event_ts_us 线性锚定到手机钟;
3. 实测(真我 GT8):IMU ~987Hz(三传感器合计),事件 9.5M ev/s,视频 ~2.6MB/s,
   三路并发录制时预览仍 30fps。

### 回放事件文件

1. 把 .raw 文件放进手机 `/sdcard/evk_data/`;
2. App 首次使用需授予「所有文件访问」权限;
3. 「事件文件」卡片点「播放」:预览 + 统计 + 进度条拖动。

> seek 实现要点:EVT3 是状态流(时间增量编码 + 向量/地址上下文),跳转字节偏移后必须
> 调 `I_EventsStreamDecoder::reset_last_timestamp(bookmark_ts)` 重建解码器时间基准,
> 否则跳转后事件时间戳错乱、节拍失效(表现为拖动/重播后画面异常)。

### RGB 变焦

「RGB 预览」卡片支持**数字变焦**:预览区双指捏合或拖动「变焦」滑条(倍率显示在标题
右侧,范围来自设备 ZoomState)。变焦同时作用于预览和录制的 MP4——更换不同焦距的
事件相机镜头后,可用它匹配 RGB 与事件相机的视场。

### 坏点屏蔽(Digital Event Mask)

「采集控制」卡片底部的「坏点屏蔽」区,用 HAL 的 `I_DigitalEventMask` 在**传感器数字
管线内**屏蔽坏点/热像素——被屏蔽的像素不会产生事件,**raw、预览、统计同时干净**
(不同于主机端软件过滤:后者 raw 里照旧有坏点,只是不显示)。用法:

1. 开关默认**开启**,内置本机实测的 3 个坏点(`578,110` / `250,628` / `659,704`,
   持续振荡、合计约吃掉 3M ev/s 的带宽与 ERC 预算);
2. 输入框填 `x,y` 点「添加」即屏蔽;下方芯片列出已屏蔽点,点 ✕ 删单个;
   「恢复默认」回到内置 3 点,「清空」全部解除;
3. 改动**实时生效**(无需重连),列表与开关持久化保存(SharedPreferences);
   **录制中锁定编辑**,保证 metadata 与实际一致;
4. 录制元数据 `event_camera.masked_hotpixels` 记录本次生效的屏蔽点(对齐 PC 端
   record_event_imu_session.py 的字段)。

实现要点(见 `native/evk-jni/evk_camera_jni.cpp` 的 `applyPixelMask`):
- 本机(IMX646/Gen4.1)有 **64 个掩码槽位**(`get_pixel_masks().size()`),超出截断;
- 每次 `set_mask()` = 3 个寄存器域的读改写 ≈ 6 个 USB 控制帧,数点毫秒级;
- 掩码在事件流 start/stop 间保持,但**设备关闭后丢失** → App 在每次打开相机时重新下发;
- 文件回放设备无此 facility(传感器级功能),JNI 判空跳过、UI 提示;
- 无需重编 OpenEB:该 facility 已在现成的 `libmetavision_psee_hw_layer.so` /
  `libhal_plugin_prophesee.so` 中。

> **Android RTTI 补丁(③)**:`I_DigitalEventMask` 原本是内联虚析构(无 key function),
> typeinfo 为弱符号且随 `RTLD_LOCAL` 插件各自一份,导致 `Device::get_facility` 的
> `dynamic_cast` 跨库失败、facility 恒为 null(功能静默失效)。修法与 ①② 相同:补外置
> 析构做 key function(见 `i_digital_event_mask.h` +
> `hal/cpp/src/facilities/evk_template_instantiations.cpp`),使 typeinfo 只在
> `libmetavision_hal.so` 中强定义一份。**此条已在 `third_party/openeb` 中生效,重编
> OpenEB 会保留**(补丁随源码走,不是构建期 hack)。

> **实测验证(2026-09-22,真我 GT8)**:`digital_event_mask: 64 slots` 正常识别;
> 录两段对比(各取前 200MB,PC 端 `count_raw_hotpixels.py` 统计):
> 屏蔽开启时 3 个点 **0/0/0** 事件,关闭屏蔽后 **674/44/254** 事件——屏蔽精准、
> 且被屏蔽像素的事件确实"不进入 raw"。
> 注意:坏点活跃度随场次剧烈波动(PC 端见过占满整帧的 94~99%,也有 0.4% 和本次的
> 0.002%),遇到"某几个像素占满画面"的录制数据时,把坐标填入屏蔽列表即可源头规避。

> 更换事件相机镜头不影响坏点(坏点在传感器上),但若换用另一台相机机身,可用
> Metavision Studio 或录制数据分析工具重新确定坏点坐标后填入。

## 已知限制 / 风险

- **实况采集已实测打通**(2026-09-19,真我 GT8 + USB 扩展坞):IMX646/EVT3/1280x720,
  ERC 10M ev/s 下事件率 ~9.5M ev/s、USB2 带宽 ~317Mbps、预览 30fps、录制 ~35MB/s 均稳定。
  EVK5 供电依赖扩展坞;若相机反复掉线重枚举(dumpsys usb 可见地址 005→006→007 变化),
  优先检查坞的独立供电。
- USB2 口带宽 ~480Mbps,EVT3 下事件率上限数十 Mev/s,高动态场景务必开 ERC;
  USB3 手机可关 ERC 跑满。
- 首次连接时若 EVK5 的 VID/PID 不在 OpenEB 已知列表(04b4/03fd/1fc9),注入路径
  已放宽匹配,但仍需 logcat(tag: metavision)确认走的是 imx646 设备构建序列。
- 录制文件写入与事件解码在同一个采集线程,极端事件率下预览可能掉帧(录制不受影响)。

## 排查

```
adb logcat -s evkcam metavision
```
