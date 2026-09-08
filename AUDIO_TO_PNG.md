# 音频转 PNG 原理（yituPicBGM）

> 把音频文件无损编码进一张 PNG 图片，在只支持图片存储的平台上（如 Owlbear Rodeo 枭熊）"存放"音频；需要时下载图片，还原字节播放。本文总结其编码/解码逻辑。

## 一、为什么这么做

枭熊（Owlbear Rodeo）是一款 VTT 跑团平台，道具（token）本质是图片，本身不直接支持上传音频。但图片可以携带任意像素数据，因此可以把音频的**原始字节**写进 PNG 的像素里，把 PNG 当作一个"字节容器"上传。播放时由扩展把像素重新读回字节，交给浏览器 `<audio>` 播放。

## 二、编码格式（字节布局）

编码后的字节流按顺序依次为：

| 偏移 | 长度 | 含义 |
| --- | --- | --- |
| 0 | 8 字节 | 魔数 `MP3IMG01`（`0x4D 50 33 49 4D 47 30 31`） |
| 8 | 8 字节 | 音频数据长度（大端 uint64） |
| 16 | 2 字节 | 文件名字节长度（大端 uint16，UTF-8） |
| 18 | N 字节 | 文件名（UTF-8） |
| 18+N | M 字节 | 原始音频字节（如 MP3 文件内容） |
| … | 填充 | `0x00` 补齐到 3 的整数倍（对齐像素） |

### 像素映射

- PNG 使用 **RGB** 模式，**每个像素 3 字节**，正好容纳 3 个原始字节。
- 宽度固定（默认 `2048` 像素），高度 = `ceil(总字节数 / 3 / 宽度)`。
- 不足一行的剩余像素用 `0x00` 填满。

## 三、编码流程（Python 端）

对应文件：`mp3img.py`（独立 CLI）与 `app.py`（本地转换网页服务，同一套算法）。

1. 读入音频文件全部字节 `data`。
2. 构造头部：魔数 + 数据长度 + 文件名长度 + 文件名。
3. 拼接 `header + data`，用 `0x00` 补齐到 3 的整数倍。
4. 计算像素数 `n_pixels = len(payload) // 3`，确定图片高度。
5. `Image.frombytes("RGB", (width, height), payload)` 生成图片。
6. 以 **PNG（无损）** 保存。

关键：PNG 本身无损，编码后体积约等于原始音频（DEFLATE 压缩后甚至略小，接近 0% 额外开销）。

## 四、解码流程（浏览器端）

对应文件：`audio-play/src/audioCodec.ts`（扩展内）。

1. `fetch(url)` 下载 PNG → `createImageBitmap` 解码。
2. 画到 `<canvas>`，`getImageData` 取出像素（浏览器给出 **RGBA**，每像素 4 字节）。
3. 每 4 字节只取 R/G/B（跳过 alpha），重建 RGB 字节流。
4. 校验前 8 字节魔数是否等于 `MP3IMG01`，不匹配则报错（图片被压缩/修改过）。
5. 读数据长度、文件名长度、文件名。
6. 截取 `[18+nameLen, 18+nameLen+dataLen)` 得到原始音频字节。
7. `new Blob([bytes], {type:"audio/mpeg"})` → `URL.createObjectURL` → `new Audio(url)` 播放。

## 五、约束与注意事项

- **必须用 PNG**，严禁 JPEG/WebP 等有损格式——有损压缩会改变像素值，导致字节损坏、解码失败。
- 上传平台时**必须关闭**任何"压缩 / 转格式 / Storage Saver / 缩放"选项，否则像素被改。
- 文件名 UTF-8 编码，最长 65535 字节。
- 编码/解码两端魔数、字节序（大端）必须严格一致（`mp3img.py` ↔ `audioCodec.ts` 已对齐）。
- 每个音频的 PNG 大小 ≈ 音频大小，受枭熊存储档位限制（Nestling/Fledgling 25MB/图，Bestling 50MB/图）。

## 六、与 Owlbear Rodeo 的配合

1. 本地用 `mp3img.py`（或 `音频转PNG.exe`）把音频转成 PNG 噪点图。
2. 把 PNG 上传为枭熊道具（token）。
3. 扩展 `yituPicBGM` 扫描场景道具，通过 `item.image.url` 下载 PNG、解码还原音频，在房间内播放（GM 控制、多端同步）。

## 相关文件

- 编码：`mp3img.py`、`app.py`
- 解码/播放：`audio-play/src/audioCodec.ts`、`audio-play/src/player.ts`
- 扩展清单：`audio-play/public/manifest.json`
