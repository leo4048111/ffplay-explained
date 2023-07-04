# ffplay.c源码分析与理解

## 前言

### ffplay定义

**ffplay是FFmpeg提供的一个极为简单的音视频媒体播放器（由ffmpeg库和SDL库开发），可以用于音视频播放、可视化分析 ，提供音视频显示和播放相关的图像信息、音频的波形等信息。**

### 版本信息

**本文的ffplay源码分析基于 Jul 3, 2023，commit 50f34172e0cca2cabc5836308ec66dbf93f5f2a3的最新ffplay.c源码版本。**

## 架构分析

**从ffplay.c源码中粗略分析，我发现当前ffplay播放器的架构由4种类型的线程构成，所有的线程类型和其相应的功能描述如下：**

1. **SDL窗口主线程（main）：**该线程是ffplay程序的主线程，以main函数为入口，首先初始化了ffmpeg和SDL上下文，随后调用`stream_open`函数打开输入文件/流，并且完成相关的子线程创建逻辑。最后，进入SDL窗口的`event_loop`循环。在每次循环中，先通过`refresh_loop_wait_event`执行图像渲染逻辑，随后在`switch(event.type)`中处理SDL窗口消息，实现鼠标和键盘控制全屏、暂停播放、继续播放、退出等人机交互功能。
2. **解复用线程（read_thread）：**该线程由主线程调用的`stream_open`中的`is->read_tid   = SDL_CreateThread(read_thread, "read_thread", is);`一句创建。解复用线程启动后，首先通过ffmpeg提供的`avformat_open_input(...)`方法打开输入流/文件。随后，设置相关扫描参数后，调用`avformat_find_stream_info`确认输入流/文件是否含有有效的stream信息。紧接着，解复用线程通过`av_find_best_stream`方法，找到音频、视频和字幕（此处不做讨论）对应的流索引，存放在`st_index[AVMEDIA_TYPE_NB]`数组中。随后，使用`stream_component_open`打开每个流，为每个流创建对应的解码线程（详述见下）。在上面的步骤都完成后，解复用线程进入自己的线程循环，每次循环中的主要工作就是通过`av_read_frame`从输入流/文件中读出一个`AVPacket`，然后根据这个`pkt`的`pkt->stream_index`，调用`packet_queue_put`把它缓存到每个stream对应的`PacketQueue`中，等待在`decoder_decode_frame`函数调用中被`if (packet_queue_get(d->queue, d->pkt, 1, &d->pkt_serial) < 0)`一句拿出然后送到解码器去解码（`PacketQueue`结构体的分析见下文）。
