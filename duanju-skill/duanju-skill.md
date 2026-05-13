---
name: duanju
description: 短视频引流解说视频制作技能包。适用场景：AI漫剧 / 短剧 / 爽文视频 的60-90秒爆款引流视频（微信视频号、抖音）。基于 video-use 主技能扩展，5步流程：转录→EDL→裸片→配音+校准SRT→合成输出。核心规则：配音必须先转录再写字幕，禁止估算时间戳。
---

# duanju-skill

短视频引流解说视频制作技能包

## 适用场景

AI漫剧 / 短剧 / 爽文视频 的60-90秒爆款引流视频，目标平台为微信视频号、抖音等竖屏短视频平台。核心KPI：完播率>40%，点赞率>5%。

## 核心流程（5步走）

```
Step 1  转录素材  → takes_packed.md（词级时间戳）
Step 2  构建 EDL  → edl.json（剪辑决策）
Step 3  渲染视频  → base.mp4（无字幕裸片）
Step 4  生成配音  → narration.mp3 + 精准SRT
         ⚠️ 必须先转录配音获取词级时间戳，再逐句写SRT
Step 5  合成输出  → final.mp4（配音+BGM+字幕+CTA）
```

---

## 关键经验教训（必读，执行前背熟）

### 🔴 教训1：字幕时间必须用实际转录，禁止估算

**错误做法：** 根据配音时长估算每句字幕的时间点
**后果：** 字幕和配音完全对不上，差几秒，用户直接弃看

**正确做法：**
1. 先生成配音 MP3
2. 用 ElevenLabs Scribe 转录配音，得到词级时间戳
3. 根据实际时间戳逐句写 SRT

```bash
# 步骤：先生成配音，再转录配音
python generate_narration.py  # 先生成配音
python helpers/transcribe.py narration.mp3 --edit-dir ./edit  # 转录配音
# 读取 transcripts/narration.json 逐句校准 SRT
```

### 🔴 教训2：render.py 在 Windows 必须加 encoding='utf-8'

```python
# ❌ 错误（GBK环境报错）
edl = json.loads(edl_path.read_text())

# ✅ 正确（全部加 encoding='utf-8'）
edl = json.loads(edl_path.read_text(encoding='utf-8'))
concat_list.write_text("...", encoding='utf-8')
out_path.write_text("...", encoding='utf-8')
transcript = json.loads(tr_path.read_text(encoding='utf-8'))
```

### 🔴 教训3：Windows ffmpeg 路径冒号要转义

```bash
# ❌ 错误（冒号被解析为选项）
-vf "subtitles='e:/Desktop/漫剧/edit/explosive_subs.srt'"

# ✅ 正确（反斜杠转义冒号）
-vf "subtitles='e\\:/Desktop/漫剧/edit/explosive_subs.srt'"
```

---

## 每步详细操作

### Step 1：转录素材

```bash
# 批量转录（首次）
python helpers/transcribe_batch.py "e:/源视频目录/" --edit-dir "e:/输出目录/edit" --workers 4

# 打包为 takes_packed.md（后续从这里读爆点）
python helpers/pack_transcripts.py --edit-dir "e:/输出目录/edit"
```

### Step 2：构建 EDL

参考格式 `edl.json`：

```json
{
  "version": 1,
  "sources": {
    "第1集.mp4": "e:/源视频目录/第1集.mp4",
    "第8集.mp4": "e:/源视频目录/第8集.mp4"
  },
  "ranges": [
    {"source": "第1集.mp4", "start": 25.00, "end": 25.33, "beat": "hook_flash_01", "reason": "房产证展示 最高光开场帧"},
    {"source": "第8集.mp4", "start": 12.00, "end": 20.00, "beat": "fight_02", "reason": "压缩至8秒"}
  ],
  "grade": "none",
  "overlays": [],
  "total_duration_s": 82.71
}
```

**视频结构参考（83秒总时长）：**
- 0-5s：核弹级闪回蒙太奇（15帧×0.33s）
- 5-20s：痛点（被占房、被嘲讽、丈夫偏心）
- 20-50s：高光（反击、名场面、霸气台词）
- 50-83s：悬念结尾+引流CTA

### Step 3：渲染裸片

```bash
cd video-use-main
python helpers/render.py "e:/输出目录/edit/edl.json" \
  -o "e:/输出目录/edit/base.mp4" \
  --no-loudnorm
```

### Step 4：生成配音 + 精准字幕（关键步骤）

**① 生成配音脚本 `generate_narration.py`：**

```python
import asyncio, edge_tts
from pathlib import Path

NARRATION_TEXT = """配音文案，控制总时长在视频时长的60%左右
语速快的短剧风格，每句话简短有力"""

async def main():
    out = Path("e:/输出目录/edit/narration.mp3")
    # 热血男声，rate +30%
    communicate = edge_tts.Communicate(NARRATION_TEXT, "zh-CN-YunjianNeural", rate="+30%")
    await communicate.save(str(out))

asyncio.run(main())
```

**② 转录配音获取精确时间戳：**

```bash
python helpers/transcribe.py "e:/输出目录/edit/narration.mp3" --edit-dir "e:/输出目录/edit"
```

**③ 根据 `transcripts/narration.json` 逐句写 SRT：**

从 narration.json 读取每个词的 start/end 时间，按人声自然停顿断句，每行0.5-3秒。

**SRT 格式示例：**
```
1
00:00:00,280 --> 00:00:01,240
被全网吹爆的

2
00:00:01,240 --> 00:00:02,339
AI漫剧！
```

### Step 5：合成最终视频

```bash
# ① 配音混入原剧音频（配音为主，原剧背景音为辅）
ffmpeg -y \
  -i "e:/输出目录/edit/base.mp4" \
  -i "e:/输出目录/edit/narration.mp3" \
  -filter_complex "[1:a]volume=1.0[nar];[0:a]volume=0.25[bga];[nar][bga]amix=inputs=2:duration=longest:weights=1 0.25[out]" \
  -map 0:v -map "[out]" -c:v copy -c:a aac -b:a 192k -ar 48000 \
  "e:/输出目录/edit/preview_audio.mp4"

# ② 烧入字幕（Windows路径转义冒号）
ffmpeg -y \
  -i "e:/输出目录/edit/preview_audio.mp4" \
  -vf "subtitles='e\\:/输出目录/edit/explosive_subs.srt':force_style='FontName=微软雅黑,FontSize=18,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,Outline=1,Shadow=0,Alignment=2,MarginV=100'" \
  -c:v libx264 -preset fast -crf 20 -pix_fmt yuv420p -c:a copy \
  "e:/输出目录/edit/preview_subtitled.mp4"

# ③ 添加顶部CTA绿色大字（全程显示）
ffmpeg -y \
  -i "e:/输出目录/edit/preview_subtitled.mp4" \
  -i "e:/输出目录/edit/cta_overlay.png" \
  -filter_complex "[0:v][1:v]overlay=0:0:enable='gte(t,0)'" \
  -c:v libx264 -preset fast -crf 20 -pix_fmt yuv420p -c:a copy \
  "e:/输出目录/edit/final.mp4"
```

---

## CTA大字条制作（PIL）

绿色文字，无背景，全屏透明PNG，全程overlay显示。

```python
from PIL import Image, ImageDraw, ImageFont
from pathlib import Path

W, H = 1080, 1920
cta_text = "点击左下角 看全集更新！"
font_size = 64  # 比字幕18pt大得多
font = ImageFont.truetype("C:/Windows/Fonts/msyh.ttc", font_size)

img = Image.new("RGBA", (W, H), (0, 0, 0, 0))
draw = ImageDraw.Draw(img)
bbox = draw.textbbox((0, 0), cta_text, font=font)
text_w = bbox[2] - bbox[0]
x = (W - text_w) // 2 - bbox[0]
y = 40 - bbox[1]  # 顶部

# 绿色描边
for dx in range(-3, 4):
    for dy in range(-3, 4):
        if dx != 0 or dy != 0:
            draw.text((x + dx, y + dy), cta_text, font=font, fill=(0, 150, 0, 255))
draw.text((x, y), cta_text, font=font, fill=(0, 255, 80, 255))

img.save("e:/输出目录/edit/cta_overlay.png", "PNG")
```

---

## 常用命令速查

| 需求 | 命令 |
|---|---|
| 转录多集 | `python helpers/transcribe_batch.py "源目录/" --edit-dir "输出目录/edit" --workers 4` |
| 渲染视频 | `python helpers/render.py edl.json -o out.mp4 --preview --no-loudnorm` |
| 检查时长 | `ffprobe -v quiet -show_entries format=duration -of csv=p=0 video.mp4` |
| 生成配音 | `python generate_narration.py` |
| 转录配音 | `python helpers/transcribe.py narration.mp3 --edit-dir ./edit` |
| 提取帧 | `ffmpeg -y -ss 5 -i video.mp4 -frames:v 1 out.jpg` |

---

## 字幕样式推荐

| 场景 | force_style 参数 |
|---|---|
| 白色干净体（当前用） | `FontName=微软雅黑,FontSize=18,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,Outline=1,Shadow=0,Alignment=2,MarginV=100` |
| 爆款黄字 | `FontName=微软雅黑,FontSize=18,Bold=1,PrimaryColour=&H00FFFF00,OutlineColour=&H00000000,Outline=3,Shadow=0,Alignment=2,MarginV=80` |
| 底部小字幕 | `FontName=微软雅黑,FontSize=14,Bold=0,PrimaryColour=&H00CCCCCC,OutlineColour=&H00000000,Outline=1,Shadow=0,Alignment=2,MarginV=120` |

---

## 文件命名规范

```
edit/
├── edl.json                  ← 剪辑决策
├── base.mp4                  ← 裸片（无字幕）
├── preview_audio.mp4         ← 加了配音的版本
├── preview_subtitled.mp4     ← 加了字幕的版本
├── final.mp4                 ← 最终输出
├── narration.mp3             ← AI配音原文件
├── explosive_subs.srt        ← 精准字幕（从配音转录校准）
├── cta_overlay.png           ← 顶部CTA大字
├── generate_narration.py     ← 配音生成脚本
├── clips_preview/            ← 片段提取目录
├── transcripts/
│   ├── 源视频-第1集.json
│   └── narration.json        ← 配音转录（关键！）
├── takes_packed.md           ← 所有转录打包
└── verify/                   ← QC截图目录
```

---

## 快捷启动（复制粘贴用）

```bash
# 完整流程

# 1. 转录
python helpers/transcribe_batch.py "./src/" --edit-dir "./edit" --workers 4
python helpers/pack_transcripts.py --edit-dir "./edit"

# 2. 渲染裸片
python helpers/render.py "./edit/edl.json" -o "./edit/base.mp4" --no-loudnorm

# 3. 生成配音
python "./edit/generate_narration.py"

# 4. 转录配音获取时间戳
python helpers/transcribe.py "./edit/narration.mp3" --edit-dir "./edit"

# 5. 根据 narration.json 校准 explosive_subs.srt

# 6. 合成
ffmpeg -y -i "./edit/base.mp4" -i "./edit/narration.mp3" \
  -filter_complex "[1:a]volume=1.0[nar];[0:a]volume=0.25[bga];[nar][bga]amix=inputs=2:duration=longest:weights=1 0.25[out]" \
  -map 0:v -map "[out]" -c:v copy -c:a aac -b:a 192k -ar 48000 \
  "./edit/preview_audio.mp4"

ffmpeg -y \
  -i "./edit/preview_audio.mp4" \
  -vf "subtitles='e\\:/你的路径/edit/explosive_subs.srt':force_style='FontName=微软雅黑,FontSize=18,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,Outline=1,Shadow=0,Alignment=2,MarginV=100'" \
  -c:v libx264 -preset fast -crf 20 -pix_fmt yuv420p -c:a copy \
  "./edit/preview_subtitled.mp4"

ffmpeg -y \
  -i "./edit/preview_subtitled.mp4" \
  -i "./edit/cta_overlay.png" \
  -filter_complex "[0:v][1:v]overlay=0:0:enable='gte(t,0)'" \
  -c:v libx264 -preset fast -crf 20 -pix_fmt yuv420p -c:a copy \
  "./edit/final.mp4"
```

---

## 项目记忆（错位屋檐案例参数）

| 参数 | 值 |
|---|---|
| 源视频 | 15集，1080×1920，30fps，H.264+AAC |
| 总时长 | 83秒（目标60-90秒 ✅） |
| 配音 | zh-CN-YunjianNeural，rate +30%，总长48.9s |
| 配音转录 | 245词，词级时间戳 |
| SRT行数 | 32行（跟着人声节奏断句） |
| 字幕字体 | 微软雅黑 18pt，白色，MarginV=100 |
| CTA大字 | 微软雅黑 64pt，绿色(0,255,80)，无背景，顶部居中 |
| 音频混合 | 配音100% + 原剧25% |
| 成品路径 | `e:/Desktop/漫剧/错位屋檐/edit/preview_final.mp4` |

---

**记住：配音必须先转录再写字幕，时间戳用实际数据不用估算。**