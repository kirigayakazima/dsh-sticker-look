# dsh-sticker-look

给纯文本主模型的「看图」工具。补的不是"看见"，是**看懂**。

- 动图先抽帧拼联系表，再交给视觉模型——GIF 进宿主只会剩第一帧，把「踢到脚疼得跳」看成「小跑」就是这么来的。
- 库里有的图，以她写的约定为准；库里没有的，**不许直接下结论**，只给候选并要语境。
- 画面上的表情 ≠ 对方要表达的情绪。同一张低头望窗外的图，有人用来表示开心。
- 联系表每格左上角画帧号（点阵数字，不依赖字体文件），方便对时长表；v0.1 漏了这个。

零运行时依赖：GIF 解码、联系表、PNG 编码都是自己实现的（只用 `node:zlib`），不依赖 sharp / omggif / pngjs / ffmpeg，也不需要系统装任何东西。

## 装

```bash
cp -a /path/to/sticker-look /root/sticker-look
dsh plugin --profile web add link:/root/sticker-look
```

`@deepseek-ai/*` 由宿主树解析（实测 0.2.0-rc.2 有自定义 resolver），不用往 profile 里塞 node_modules。

配置写进 profile 的 `cordis.patch.yml`：

```yaml
- id: sticker-look
  config:
    vision:
      provider: deepseek-official
      model: deepseek-flash     # 这个 adapter 的 inputModalities 要含 image
      maxTokens: 3000           # 会思考的模型给小了会「没返回文字」
    library: ~/.dsh/stickers
```

改完重启 dsh web。

## 用

| 参数 | 说明 |
|---|---|
| `image` | 本地图路径（gif/png/jpg/webp） |
| `context` | **最有用的一项**：他发这张时的语境，比如"刚聊完一件开心的事，他说在下班路上" |
| `question` | 她想问什么 |
| `cheap` | `true` = 只查库里的约定，不调模型 |
| `frames` | 动图抽几帧，默认 9 |

## 库的格式

`library` 目录下任意 md 里一行一条，插件按文件名索引：

```
- run.gif —— 踢桌腿上的水瓶，踢到桌角，抱着腿疼得直跳。倒霉、翻车、工具报错。
- http-404.jpg —— 空电饭煲，满地问号。看不懂、找不到、他说的我没接上。
```

`——`、`—`、`--` 都认。库就是这套约定的唯一权威：图上没有的意思，只有这里能记。

## 输出

三段：**画面**（只写看得见的，图上的字逐字抄）、**意思**（1–2 个候选 + 能确定/不能确定分开写）、**一句话**（语境够了就直接给结论，别用「拿不准」糊过去；语境不够才写「拿不准：A 或 B」）。

`cheap: true` 时只回库里的原话，不产生模型调用。

## 为什么库里没有就不许下结论

真机踩过的坑：一张「低头 + 望窗外」的图，视觉模型读成「委屈、emo、让我自己待会儿」，而对方用它表示**开心**——他也是在下班班车上发的，顺便说「这是我的心情，表示很开心」。

含义不在画面里，在**时机**里。所以插件对没有约定的图只给候选，并明确要求补 `context`；有了 `context`（刚聊完开心的事、在路上）才允许收敛。

## 自检

```bash
node test/selfcheck.mjs
# 指定别的库：LIB=/path/to/stickers node test/selfcheck.mjs
```

覆盖：解码（帧数/尺寸/逐帧时长）、透明索引回归、采样、联系表、提示词两种模式、返回值不含 `undefined`（宿主会判 invalid output）、解码层无外部依赖。

测试里有一张手工构造的 1x1 GIF，专门钉住 GCE 透明索引的读取位置——读错一位，真实表情包的对应帧会整块变黑。

## 已知限制

- 动图 WebP / APNG 只给第一帧（宿主图像库本身解不了），提示词里会说明。
- 视频（mp4/webm）不做。
- 视觉模型仍可能编梗；提示词只能压住"没语境下结论"，压不住它硬猜。
- 联系表默认 9 帧、长边 320，会漏掉两格之间的短暂动作。

MIT
