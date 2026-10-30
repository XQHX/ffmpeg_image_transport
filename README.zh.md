# ffmpeg_image_transport（中文说明）

`image_transport` 的 ffmpeg 插件，注册名 **`ffmpeg`**。发布端把 `sensor_msgs/Image` 编码成 `FFMPEGPacket` 发到 `<话题>/ffmpeg`。本分支当前只完成编码端。

## 1. 依赖：必须配套这个 encoder 分支

```bash
git clone https://github.com/LHiou/ffmpeg_encoder_decoder.git
```

本包调用的 `setUseRga()`、`setEncoderColorSpace()`、`setEncoderColorRange()`、`setRgaCores()` 以及 NV12 直通编码只存在于该分支，换成上游或 apt 版本会**编译失败**（不是只缺参数）。`package.xml` 的 `<depend>` 只决定编译顺序、不拉源码，两个包放进同一工作空间即可，colcon 会自动先编 encoder。

其余：`ffmpeg_image_transport_msgs`、`image_transport`、`rclcpp`、`cv_bridge`；FFmpeg（`libav*`）；`librga`（可选，供 `encoder_use_rga`）。

## 2. 编码（发布端）

插件不是独立进程，也没有自己的 launch 文件。业务节点用 image_transport 的标准 API 发布，插件在节点里被动态加载，参数按节点配置。

### 2.1 发布

```cpp
#include <image_transport/image_transport.hpp>

class CameraNode : public rclcpp::Node
{
public:
  CameraNode() : Node("camera_node")
  {
    // advertise /camera0/image (raw) 和 /camera0/image/ffmpeg (packet)
    pub_ = image_transport::create_publisher(
      this, "camera0/image", rclcpp::QoS(40).reliable().get_rmw_qos_profile());
  }

  void onFrame(const sensor_msgs::msg::Image & img)
  {
    pub_->publish(img);   // encoding: bgr8 / rgb8 / nv12
  }

private:
  image_transport::Publisher pub_;
};
```

- 必须用 `image_transport::create_publisher`；直接 `create_publisher<sensor_msgs::msg::Image>` 只有 raw 话题，插件不参与。
- 编码器在**第一次 publish** 时按图像尺寸初始化，参数必须在创建 publisher 之前就位；之后再改无效，要重建 publisher 或重启节点。
- 只收压缩数据的一端订阅 `<话题>/ffmpeg`。

### 2.2 参数

插件没有自己的配置文件，参数都声明在它所属的节点上。参数前缀由**话题名**决定：取话题全名，去掉节点命名空间，`/` 换成 `.`，末尾加 `.ffmpeg.`。插件初始化时按这个前缀读参数，所以每路话题可以各配一套：

| 节点命名空间 | 发布的话题 | 插件读取的参数前缀 |
|---|---|---|
| `/` | `camera0/image` | `camera0.image.ffmpeg.*` |
| `/camera0` | `image` | `image.ffmpeg.*` |

以节点在 `/camera0` 下发布 `image` 为例，下面 YAML 的第一行就是它的编码器参数：

```yaml
# config/encoder.yaml
/**:
  ros__parameters:
    image.ffmpeg.encoder: h264_rkmpp
    image.ffmpeg.frame_rate: 20
    image.ffmpeg.bit_rate: 2000000
    image.ffmpeg.gop_size: 20
    image.ffmpeg.max_b_frames: 0
    image.ffmpeg.pixel_format: nv12
    image.ffmpeg.encoder_av_options: "rc_mode:CBR"
    image.ffmpeg.encoder_color_space: bt601
    image.ffmpeg.encoder_color_range: limited
    image.ffmpeg.encoder_use_rga: true
    image.ffmpeg.rga_cores: 3
```

```python
Node(
    package="your_camera_pkg", executable="camera_node",
    namespace="camera0",
    parameters=[os.path.join(
        get_package_share_directory("your_camera_pkg"), "config", "encoder.yaml")],
)
```

- 多相机：每路一个命名空间（`/camera0`、`/camera1`…），一份 YAML 通吃。
- 同一个节点里发多路图：按 `camera0.image.ffmpeg.*`、`camera1.image.ffmpeg.*` 分别写。
- 参数只写在 YAML 里，launch 只加载文件，不另外传参。

### 2.3 验证

```bash
ros2 run image_transport list_transports      # 应出现 image_transport/ffmpeg
ros2 param list /camera0/camera_node | grep 'image.ffmpeg'
ros2 topic hz /camera0/image/ffmpeg           # 编码帧率
ros2 param dump /camera0/camera_node          # 参数实际取值
```

## 3. 编码参数

前缀见上，下表只写后半段。

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `frame_rate` | int | 100 | 编码帧率，**必须与相机实际帧率一致**（如 20）。上游固定 100，码率控制会按错误帧率分配每帧比特。 |
| `encoder_use_rga` | bool | `false` | 用 librga/im2d 做 `BGR24/RGB24 → NV12`，替代 CPU `sws_scale`。要求输入 `bgr8`/`rgb8`。 |
| `rga_cores` | int | 3 | RGA3 核心：0 自动、1 core0、2 core1、3 两个都用。地址受限时用 1/2。 |
| `encoder_color_space` | string | `bt601` | RGB→YUV 矩阵，`bt601`/`bt709`。 |
| `encoder_color_range` | string | `full` | YUV 范围，`full`/`limited`。 |
| `encoder` | string | `libx264` | 编码器名：`h264_rkmpp`、`libx264`、`hevc_nvenc`… |
| `bit_rate` | int | -1 | 目标码率 bit/s，-1 用编码器默认。 |
| `gop_size` | int | 10 | 关键帧间隔，越小延迟越低；不设置会告警。 |
| `max_b_frames` | int | 0 | B 帧数，硬件编码保持 0。 |
| `pixel_format` | string | "" | `av_source_pixel_format`，如 `nv12`、`yuv420p`；空则用编码器首选。 |
| `qmax` | int | -1 | 最大量化步长。 |
| `encoder_av_options` | string | "" | 透传 libav 选项，`key:value,key:value`，值里不能再含冒号。 |
| `cv_bridge_target_format` | string | `bgr8` | 首次转换目标格式；Bayer 源可设 `bayer_rggb8` 省掉 CPU 去马赛克。 |
| `encoder_measure_performance` | bool | `false` | 打印各阶段耗时。 |

## 4. 输入格式分支

按消息的 `encoding` 自动选路，不需要额外参数：

- `bgr8`/`rgb8`：cv_bridge 转 `cv::Mat` → `encoder_use_rga=true` 时 RGA 转 NV12，否则 `sws_scale`；
- `nv12`：跳过 cv_bridge，按 `step` 直接填 `AVFrame`，省一次拷贝和一次转换。要求宽高与 `step` 为偶数、平面连续，编码器像素格式为 `nv12`（设 `pixel_format: nv12`）。

`encoder_color_space`/`encoder_color_range` 只在 RGA 转换或 NV12 直通时写入编码上下文。

## 5. 解码（未完成）

当前分支只完成了编码端，解码尚未完成。订阅端的 `ffmpeg` 插件仍由上游代码提供，但本分支未验证，也未加入 RGA 转换；地面端的解码方式和参数后续单独说明。

## 6. 约束与排查

- RGA 只支持 `BGR24/RGB24 ↔ NV12`，尺寸必须一致，NV12 宽高必须为偶数；失败直接抛异常，不回退 CPU。
- `bt709` + `full` 组合 RGA 不支持，初始化报错。编码端色彩矩阵/范围要与相机一致，否则偏色。
- 编码器在第一帧时按尺寸初始化，运行中不能改分辨率，需重启节点。
- 水平分辨率非 32 倍数时上游会告警。
- 参数不生效先查键名：`ros2 param list` 里实际叫什么，就按什么改。
- 硬件路径需容器暴露 `/dev/mpp_service`、`/dev/rga`、`/dev/dri`、`/dev/dma_heap`（按板端实际）。

## 7. 部署

容器化构建见项目侧 `Dockerfile.ffmpeg_image_transport_hard`（ffmpeg-rockchip 开 `--enable-rkmpp --enable-rkrga`，MPP/RGA 装在 `/usr/local`，再 `colcon build --merge-install`）。源码构建同样只需三个包同工作空间 + `colcon build`。

## 8. 许可

沿用上游 Apache 2.0。
