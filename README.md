# ffplay.c源码分析与理解

## 前言

### ffplay定义

**ffplay是FFmpeg提供的一个极为简单的音视频媒体播放器（由ffmpeg库和SDL库开发），可以用于音视频播放、可视化分析 ，提供音视频显示和播放相关的图像信息、音频的波形等信息。**

### 版本信息

**本文的ffplay源码分析基于 Jul 3, 2023，commit 50f34172e0cca2cabc5836308ec66dbf93f5f2a3的最新ffplay.c源码版本。限于本人技术水平有限，分析中如有谬误，欢迎提交issue批评指正！**

## 架构分析

**从ffplay.c源码中粗略分析，我发现当前ffplay播放器的架构由4种类型的线程构成，所有的线程类型和其相应的功能描述如下：**

1. **SDL窗口主线程（main）：**该线程是ffplay程序的主线程，以main函数为入口，首先初始化了ffmpeg和SDL上下文，随后调用`stream_open`函数打开输入文件/流，并且完成相关的子线程创建逻辑。最后，进入SDL窗口的`event_loop`循环。在每次循环中，先通过`refresh_loop_wait_event`执行图像渲染逻辑，随后在`switch(event.type)`中处理SDL窗口消息，实现鼠标和键盘控制全屏、暂停播放、继续播放、退出等人机交互功能。
2. **解复用线程（read_thread）：**该线程由主线程调用的`stream_open`中的`is->read_tid   = SDL_CreateThread(read_thread, "read_thread", is);`一句创建，主要负责对于输入流/文件的解复用工作，即提取`AVPacket`然后缓存到对应的`PacketQueue`中。解复用线程启动后，首先通过ffmpeg提供的`avformat_open_input(...)`方法打开输入流/文件。随后，设置相关扫描参数后，调用`avformat_find_stream_info`确认输入流/文件是否含有有效的stream信息。紧接着，解复用线程通过`av_find_best_stream`方法，找到音频、视频和字幕对应的流索引，存放在`st_index[AVMEDIA_TYPE_NB]`数组中。随后，使用`stream_component_open`打开每个流，为每个流创建对应的解码线程（详述见下）。在上面的步骤都完成后，解复用线程进入自己的线程循环，每次循环中的主要工作就是通过`av_read_frame`从输入流/文件中读出一个`AVPacket`，然后根据这个`pkt`的`pkt->stream_index`，调用`packet_queue_put`把它缓存到每个stream对应的`PacketQueue`中，等待在`decoder_decode_frame`函数调用中被`if (packet_queue_get(d->queue, d->pkt, 1, &d->pkt_serial) < 0)`一句拿出然后送到解码器去解码。（`PacketQueue`结构体分析见下文）
3. **解码线程（video_thread, audio_thread, subtitle_thread)：**这一类线程负责对于`AVPacket`的解码工作，ffplay中解码线程有`video_thread`，`audio_thread`和`subtitle_thread`三个，这里字幕解码不做讨论。这些线程都是在解复用线程`read_thread`中，由`stream_component_open`过程中的`decoder_start`函数调用创建的。笼统来看，解码线程的主体都是一个`for(;;)`循环，每次循环中的流程首先是获取一个对应的`AVFrame`视频或者音频帧（`audio_thread`中直接调用`decoder_decode_frame`获取音频帧，而`video_thread`中调用`get_video_frame`获取视频帧，它是对于`decoder_decode_frame`的封装）。随后，根据解码后的`AVFrame`中的数据，计算相关参数。最后，将`AVFrame`添加到视音频对应的`FrameQueue`队列中（`audio_thread`中调用`frame_queue_push`，而`video_thread`中调用`queue_picture`，它是对于`frame_queue_push`的封装），分别供音频播放线程和主线程在音频播放和图像渲染时使用。（`FrameQueue`结构体分析见下文）
4. **音频播放线程（sdl_audio_callback）：**该线程由解复用线程在调用`stream_component_open`打开音频流的时候，`stream_component_open`内部调用`audio_open`函数，`audio_open`内部再调用`SDL_OpenAudioDevice`创建。该线程的主要工作逻辑实现在`sdl_audio_callback`中，该回调函数会在音频播放线程中被反复调用，来向外部请求可播放的音频数据。回调函数的类型声明语句是`typedef void (SDLCALL * SDL_AudioCallback) (void *userdata, Uint8 * stream, int len)`，在音频播放线程调用的时候，会填入`len`参数来告知需要给SDL送入多少字节的数据。随后，我们只需要将需要送入的数据拷贝到`stream`指向的缓冲区即可。这里ffplay在`sdl_audio_callback`回调函数实现中，首先通过`audio_decode_frame`从音频帧的`FrameQueue`中获取一帧音频数据（调用`frame_queue_peek_readable`实现），随后通过`swr_convert`对于音频数据进行重采样，处理后的音频数据放在`is->audio_buf`中返回`sdl_audio_callback`，由回调函数逻辑调用`memcpy`将数据复制到`stream`中送给SDL进行播放。

**综上所述，ffplay整体的架构图如下所示：**
![ffplay_arch](https://github.com/leo4048111/ffplay-explained/blob/3c16fac949b361b1a6fd24331ea47bbdb3866111/ffplay_arch.png)

## 重要结构体分析

### PacketQueue

`PacketQueue`结构体的声明如下所示：

```cpp
typedef struct PacketQueue
{
    /* ffmpeg封装的队列数据结构，里面的数据对象是MyAVPacketList */
    /* 支持操作alloc2, write, read, freep */
    AVFifo *pkt_list;
    /* 队列中当前的packet数 */
    int nb_packets;
    /* 队列所有节点占用的总内存大小 */
    int size;
    /* 队列中所有节点的合计时长 */
    int64_t duration;
    /* 终止队列操作信号，用于安全快速退出播放 */
    int abort_request;
    /* 序列号，和MyAVPacketList中的序列号作用相同，但改变的时序略有不同 */
    int serial;
    /* 互斥锁，用于保护队列操作 */
    SDL_mutex *mutex;
    /* 条件变量，用于读写进程的相互通知 */
    SDL_cond *cond;
} PacketQueue;
```

其中`AVFifo *pkt_list`中存储的数据类型为`MyAVPacketList`，该结构体声明如下：

```cpp
typedef struct MyAVPacketList
{
    /* 待解码数据 */
    AVPacket *pkt;
    /* pkt序列号 */
    int serial;
} MyAVPacketList;
```

该数据结构的引入主要是为了设计⼀个多线程安全的队列，保存AVPacket，同时统计队列内已缓存的数据⼤⼩。（这个统计数据会⽤来后续设置要缓存的数据量）。同时，数据结构中引⼊了serial的概念，区别前后数据包是否连续，主要应⽤于seek操作。最后设计了两类特殊的packet——flush_pkt和nullpkt（类似⽤于多线程编程的事件模型——往队列中放⼊ flush事件、放⼊null事件），其在⾳频输出、视频输出、播放控制等模块时也会继续使用到其来队列控制
