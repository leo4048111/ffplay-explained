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

该数据结构的引入主要是为了设计⼀个多线程安全的队列，保存AVPacket，同时统计队列内已缓存的数据⼤⼩。（这个统计数据会⽤来后续设置要缓存的数据量）。同时，数据结构中引⼊了serial的概念，区别前后数据包是否连续，主要应⽤于seek操作。
对于该结构体的相关操作方法罗列如下：

+ `packet_queue_init`：用于初始化一个`PacketQueue`结构，流程上先给`pkt_list`分配内存，再创建`mutex`和`cond`变量，最后将`abort_request`设为1，这样在`stream_open`中初始化三个队列后，启动的`read_thread`里面`stream_has_enough_packets`会返回`true`，使得`read_thread`不会立即开始从输入流/文件中读取`AVPacket`，而是等待手动调用`start`启动队列后再读数据

```cpp
static int packet_queue_init(PacketQueue *q)
{
    memset(q, 0, sizeof(PacketQueue));
    q->pkt_list = av_fifo_alloc2(1, sizeof(MyAVPacketList), AV_FIFO_FLAG_AUTO_GROW);
    if (!q->pkt_list)
        return AVERROR(ENOMEM);
    q->mutex = SDL_CreateMutex();
    if (!q->mutex)
    {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateMutex(): %s\n", SDL_GetError());
        return AVERROR(ENOMEM);
    }
    q->cond = SDL_CreateCond();
    if (!q->cond)
    {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateCond(): %s\n", SDL_GetError());
        return AVERROR(ENOMEM);
    }
    q->abort_request = 1;
    return 0;
}
```

+ `packet_queue_destroy`：用于销毁一个`PacketQueue`结构，流程上先调用`packet_queue_flush`将管理的所有队列节点清除，随后释放内存然后销毁`init`里面创建的相关变量

```cpp
static void packet_queue_destroy(PacketQueue *q)
{
    packet_queue_flush(q);
    av_fifo_freep2(&q->pkt_list);
    SDL_DestroyMutex(q->mutex);
    SDL_DestroyCond(q->cond);
}
```

+ `packet_queue_start`：用于启动一个`PacketQueue`，做的工作就是把`abort_request`置0，让`read_thread`开始读数据，然后自增队列的序列号

```cpp
static void packet_queue_start(PacketQueue *q)
{
    SDL_LockMutex(q->mutex);
    q->abort_request = 0;
    q->serial++;
    SDL_UnlockMutex(q->mutex);
}
```

+ `packet_queue_abort`：用于终止一个`PacketQueue`，做的工作就是把`abort_request`置1，然后释放一个`cond`信号。这里释放`cond`的意义是，在`packet_queue_get`阻塞调用时，该函数可能会因为队列中没有数据而阻塞等待在`SDL_CondWait(q->cond, q->mutex)`这一句上。这时候如果`abort`了，读线程也不会再读入新数据，自然不会再发送新的`cond`信号来唤醒`get`的线程。所以，为了避免这种情况下线程永远阻塞，因此在`abort`时候发送一次`cond`信号，使得线程能够再运行到循环头，进入`if (q->abort_request)`的逻辑。

```cpp
static void packet_queue_abort(PacketQueue *q)
{
    SDL_LockMutex(q->mutex);
    q->abort_request = 1;
    SDL_CondSignal(q->cond);
    SDL_UnlockMutex(q->mutex);
}
```

+ `packet_queue_get`：该函数用于从一个`PacketQueue`中取出一个`pkt`。该函数的运行可能分为三种情况，对应三个返回值。首先是`av_fifo_read(q->pkt_list, &pkt1, 1) >= 0`即正确读出了一个`pkt`的情况，这时候更新相关的队列参数（这里注意`q->size`减去的是`pkt1.pkt->size + sizeof(pkt1)`，可知`q->size`算的大小是同时包括包数据和储存包的节点数据结构大小的），然后返回读出的包即可，此时返回值为1。其次，如果队列为空，并且是非阻塞运行（`packet_queue_get`调用参数`block`设为0），那么直接返回0。最后，如果队列为空且为阻塞运行(`block`为1)，则用`SDL_CondWait(q->cond, q->mutex);`入睡，等待别的线程往队列里放数据，或者`abort`时再醒过来重新执行循环

```cpp
static int packet_queue_get(PacketQueue *q, AVPacket *pkt, int block, int *serial)
{
    MyAVPacketList pkt1;
    int ret;

    SDL_LockMutex(q->mutex);

    for (;;)
    {
        if (q->abort_request)
        {
            ret = -1;
            break;
        }

        if (av_fifo_read(q->pkt_list, &pkt1, 1) >= 0)
        {
            q->nb_packets--;
            q->size -= pkt1.pkt->size + sizeof(pkt1);
            q->duration -= pkt1.pkt->duration;
            av_packet_move_ref(pkt, pkt1.pkt);
            if (serial)
                *serial = pkt1.serial;
            av_packet_free(&pkt1.pkt);
            ret = 1;
            break;
        }
        else if (!block)
        {
            ret = 0;
            break;
        }
        else
        {
            SDL_CondWait(q->cond, q->mutex);
        }
    }
    SDL_UnlockMutex(q->mutex);
    return ret;
}
```

+ `packet_queue_put`：该函数用于向`PacketQueue`中放入一个节点。在代码逻辑上，首先通过`av_packet_alloc`方法分配一个`AVPacket`，在上面的数据结构分析中我们已经可以知道这是`MyAVPacketList`中存储`Packet`数据的底层结构。随后，在分配了新的`pkt1`后，将传入的`pkt`数据引用传递给`pkt1`。随后，对于实际的`Packet`队列操作实现是在`packet_queue_put_private`函数中进行了封装，下述。

```cpp
static int packet_queue_put(PacketQueue *q, AVPacket *pkt)
{
    AVPacket *pkt1;
    int ret;

    pkt1 = av_packet_alloc();
    if (!pkt1)
    {
        av_packet_unref(pkt);
        return -1;
    }
    av_packet_move_ref(pkt1, pkt);

    SDL_LockMutex(q->mutex);
    ret = packet_queue_put_private(q, pkt1);
    SDL_UnlockMutex(q->mutex);

    if (ret < 0)
        av_packet_free(&pkt1);

    return ret;
}
```

+ `packet_queue_put_private`：该函数中封装了将`pkt`放入`PacketQueue`维护的`Packet`队列的逻辑。元素的入队采用了`av_fifo_write`进行数据拷贝，然后更新了`PacketQueue`中维护的总`pkt`个数增加1，`q->size`增加`pkt1.pkt->size + sizeof(pkt1)`（所以说这里`q->size`是队列中所有`MyAVPacketList`数据结构的大小外加其中维护的`AVPacket`数据大小。最后，将`q->duration`加上`AVPacket`的`duration`，实现了元素的入队操作。

```cpp
static int packet_queue_put_private(PacketQueue *q, AVPacket *pkt)
{
    MyAVPacketList pkt1;
    int ret;

    if (q->abort_request)
        return -1;

    pkt1.pkt = pkt;
    pkt1.serial = q->serial;

    ret = av_fifo_write(q->pkt_list, &pkt1, 1);
    if (ret < 0)
        return ret;
    q->nb_packets++;
    q->size += pkt1.pkt->size + sizeof(pkt1);
    q->duration += pkt1.pkt->duration;
    /* XXX: should duplicate packet data in DV case */
    SDL_CondSignal(q->cond);
    return 0;
}
```

+ `packet_queue_flush`：这个方法用于`flush`一个`PacketQueue`，具体原理从代码看比较简单，就是通过一个`while`循环读出队列中的所有`pkt`，逐个调用`av_packet_free`进行释放。最后，更新相关的队列参数，比如说队列中包总数、总大小、总时长等等。这里还有一个`q->serial++`的操作，具体作用将会在下文的重要变量功能分析中阐述。

```cpp
static void packet_queue_flush(PacketQueue *q)
{
    MyAVPacketList pkt1;

    SDL_LockMutex(q->mutex);
    while (av_fifo_read(q->pkt_list, &pkt1, 1) >= 0)
        av_packet_free(&pkt1.pkt);
    q->nb_packets = 0;
    q->size = 0;
    q->duration = 0;
    q->serial++;
    SDL_UnlockMutex(q->mutex);
}
```

### FrameQueue

`FrameQueue`结构体的声明如下所示：

```cpp
typedef struct FrameQueue
{
    Frame queue[FRAME_QUEUE_SIZE];
    int rindex;
    int windex;
    int size;
    int max_size;
    int keep_last;
    int rindex_shown;
    SDL_mutex *mutex;
    SDL_cond *cond;
    PacketQueue *pktq;
} FrameQueue;
```

