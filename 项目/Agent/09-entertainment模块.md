# entertainment 模块

## 模块定位

entertainment 模块（`com.yourorg.biboagent.entertainment`）是 Agent 的"娱乐中枢"：本地媒体库（音乐/视频/电子书）全生命周期管理（扫描入库、元数据补全、播放状态追踪、SSE 指令推送、AI 主动消息），并通过 14 个 `@Tool` 让 Agent 直接操控前端播放器——用户说"放首周杰伦的歌"，Agent 搜索后经 SSE 把 PLAY 指令推到浏览器执行。

---

## 一、媒体库管理

**MediaType**：MUSIC / VIDEO / BOOK。**MediaItem**（`media_items` 表）：mediaId（UUID）、type、filePath、fileSize、checksum（`fileSize:lastModified` 指纹）、metadata（JSONB，可展示元数据）、createdAt/updatedAt。**MediaMetadata**：title/author/artist/album/duration/resolution/coverSource/lyricsSource/lyricsSynced，全部 nullable（扫描时经常不完整）。

**扫描入库**（`LocalMediaScanner`）：增量扫描配置目录，用 `fileSize:lastModified` 指纹对比数据库去重（零 I/O，不打开文件）——新指纹新增，旧指纹消失标记失效。`MetadataExtractor` 提取内嵌元数据：音频 jaudiotagger（ID3 标签/封面）、视频 ffprobe（CLI）、电子书扩展名识别。`PostgresMediaLibraryService` 用 JSONB 存 metadata、ILIKE 模糊搜索；扫描末尾虚拟线程并发触发元数据补全（best-effort 不阻塞主流程）。

## 二、播放控制

**PlaybackState**（Redis `agent:playback:{userId}`，TTL 30min）：mediaId、position、duration、playing、volume、clientTimestamp、queue、mediaType。**PlaybackCommand**（AI → SSE → 前端）：command 为 PLAY/PAUSE/RESUME/SKIP + mediaId。**MediaEvent**：PLAYBACK_STARTED / PAUSED / RESUMED / SKIPPED / ENDED / LOOP_DETECTED / HALFWAY_POINT / IDLE_TIMEOUT / MILESTONE。

前端经 `POST /api/v1/entertainment/playback-state` 上报状态，`PlaybackStateService` 三步：Redis 持久化（用 clientTimestamp 比较防多标签页覆盖）→ `MediaSyncServiceImpl.evaluate()` 异步事件检测（按 userId 隔离、按 MediaType 应用频控阈值，`MIN_INTERVAL_MS = 2min`）→ `ProactiveMessageGenerator` 按事件生成中文模板消息（HALFWAY_POINT 返回 null 静默），经 SSE `PROACTIVE_MESSAGE` 推送。播放指令经 `PlaybackCommandPortImpl` 以 SSE `PLAYBACK_COMMAND` 推送到前端 `usePlaybackListener` hook 执行。

## 三、元数据在线补全

**端口设计**（六边形）：`CoverArtProvider`（实现 `ItunesCoverProvider`，600x600）；`LyricsProvider`（`LrclibLyricsProvider` 主源 + `NeteaseLyricsProvider` 回退，两步调用：搜歌→取词）；`MetadataEnrichmentService.enrich(item)` 判定缺失字段、依次尝试 provider 链、写磁盘缓存，返回 `EnrichmentResult` **但不写数据库**——持久化由调用方做（解决与 MediaLibraryService 的循环依赖）；`CoverArtCache`/`LyricsCache` 两层缓存（Caffeine 热缓存 500 条/2h + 磁盘文件冷缓存 `{mediaId}.jpg/.lrc`）。

**匹配校验**：在线返回的 artist/title 与本地元数据用 `StringSimilarity`（Jaccard token 重叠率）校验，阈值 `agent.entertainment.enrich.match-threshold`（默认 0.6）；歌词 duration 差 >15s 视为可疑跳过。封面优先提取音频内嵌封面，无内嵌才走 iTunes。

## 四、电子书系统

**入库**（`BookImportService`）：格式检测（epub/pdf/mobi/azw3/txt）→ 非 EPUB 经 `CalibreFormatConverter`（ebook-convert CLI）转换 → `EpubContentExtractor` 解析正文为 JSON 段落数组 → `BookContentIndex`（章节列表 + 段落范围/字节偏移）存 PostgreSQL `book_indexes`（chapters JSONB）→ 正文 JSON 写磁盘 `data/book-content/`。

**读取**：`FileBookContentService` 按索引字节偏移 O(1) 定位章节，分页读取（每页 1-50 段）——AI 按需读章节，不把整本书塞进上下文。

**批注**：`Annotation`（annotationId `anno_` 前缀、bookId、cfi、selectedText、note、authorType user/ai、authorId、时间戳）存 `annotations` 表；AI 通过 create/get/update/delete_annotation 四个工具操控。

## 五、AI 工具集成（14 个 @Tool）

| 组 | 工具 |
|----|------|
| 播放控制（4） | play_media、pause_playback、resume_playback、skip_track（风险 MEDIUM） |
| 队列管理（2） | add_to_queue、get_queue |
| 媒体查询（3） | search_media、get_media_info、get_library_stats |
| 批注（4） | create_annotation、get_annotations、update_annotation、delete_annotation |
| 阅读（1） | read_book_content（分页 1-50 段） |

**设计特点**：播放控制类工具**不实际播放**——后端没有音频解码能力，工具只构造 `PlaybackCommand` 经 SSE 推送到浏览器执行（"AI 大脑 → 浏览器手脚"）。工具仍走完整管线（参数校验 → 风险策略 → 执行 → 归一化 → 结果回注），结果归一化为"已向前端推送 {command} 指令，目标媒体：{title}"供 LLM 自然回复。

## 六、完整数据流

```
用户:"放首周杰伦的晴天" → LLM 决定 search_media + play_media
  → search_media 返回匹配 MediaItem
  → play_media: getById → 构造 PlaybackCommand(PLAY, mediaId) → SSE PLAYBACK_COMMAND
  → 前端 usePlaybackListener 调用播放器 API → 音乐响起
  → 前端上报 /playback-state → Redis 保存 + MediaSyncService.evaluate()
  → ProactiveMessageGenerator(PLAYBACK_STARTED) → SSE PROACTIVE_MESSAGE "《晴天》开始了，一起听吧 🎵"
```

## 七、API 端点

媒体与播放：`/api/v1/entertainment/media`（GET 列表 / POST 上传 / DELETE / PATCH rename / PATCH metadata）、`media/{id}`（GET 详情 / cover / lyrics / file 流式传输支持 HTTP Range）、`stats`、`rescan`、`playback-state`（GET/POST）、`media/{id}/cover|lyrics`（POST 手动上传 / :rematch）、`media:enrich-batch`。书籍：`/api/v1/books`（GET/POST upload）、`books/{bookId}`（GET 详情+目录）、`books/{bookId}/content/{chapter}`（GET 分页）、`books/{bookId}/annotations`（GET/POST）与 `annotations/{annotationId}`（PUT/DELETE）。

## 八、面试题整理

1. **为什么播放控制不是后端播放，而是 SSE 推指令到前端？** 后端是纯服务端，没有音频解码能力，也不该绑定播放环境。通过 SSE 推抽象 `PlaybackCommand`，后端只决策、前端适配执行——同一后端可服务 Web/移动 App/智能音箱，各端监听 PLAYBACK_COMMAND 即可。
2. **播放状态并发写入怎么保证一致性？** `RedisPlaybackStateStore.save()` 比较 clientTimestamp，只接受更新的时间戳覆盖；多标签页同时上报时旧时间戳被静默丢弃。
3. **主动消息的频控怎么做？** 按 userId 隔离，按 MediaType/事件应用阈值（MIN_INTERVAL_MS=2min）；HALFWAY_POINT 不发言；LOOP_DETECTED 有独立冷却；多层频控防刷屏。
4. **entertainment 的 @Tool 和其他模块的工具有什么区别？** 最大区别在结果生效位置：内置工具（Echo/Weather）后端执行后端返回；播放控制类工具后端决策、前端执行（SSE 推指令）——"跨端指令"模式让 Agent 能操控用户本地设备。
5. **如何防止 AI 通过播放工具执行恶意操作？** tools 模块风险策略控制（skip_track 标记 MEDIUM）；播放指令是 SSE 推送的"建议"而非强制执行，前端可实现确认弹窗或用户偏好覆盖（如夜间模式降音量）——在 AI 自主权和用户最终控制权之间平衡。
