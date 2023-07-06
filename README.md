ffplay.c源码分析与理解
=================

# 目录
* **[ffplay.c源码分析与理解](#ffplayc源码分析与理解)**
* **[目录](#目录)**
* **[前言](#前言)**
   * **[ffplay定义](#ffplay定义)**
   * **[版本信息](#版本信息)**
* **[架构分析](#架构分析)**
* **[核心结构体分析](#核心结构体分析)**
   * **[PacketQueue](#packetqueue)**
   * **[FrameQueue](#framequeue)**
   * **[VideoState](#videostate)**
   * **[Clock](#clock)**
   * **[Decoder](#decoder)**
* **[核心操作实现原理分析](#核心操作实现原理分析)**
   * **[Start(开始播放)](#start开始播放)**
   * **[Pause(暂停播放)](#pause暂停播放)**
   * **[Step(逐帧前进)](#step逐帧前进)**
   * **[Seek(跳转播放)](#seek跳转播放)**
* **[视音频同步算法原理与代码实现分析](#视音频同步算法原理与代码实现分析)**
   * **[audio_clock变量的设置时机与计算方法](#audio_clock变量的设置时机与计算方法)**
   * **[audclk变量的设置时机与计算方法](#audclk变量的设置时机与计算方法)**
   * **[音视频同步算法解析](#音视频同步算法解析)**

# 前言

## ffplay定义

**ffplay是FFmpeg提供的一个极为简单的音视频媒体播放器（由ffmpeg库和SDL库开发），可以用于音视频播放、可视化分析 ，提供音视频显示和播放相关的图像信息、音频的波形等信息。**

## 版本信息

**本文的ffplay源码分析基于 Jul 3, 2023，commit 50f34172e0cca2cabc5836308ec66dbf93f5f2a3的最新ffplay.c源码版本。限于本人技术水平有限，分析中如有谬误，欢迎提交issue批评指正！**

# 架构分析

**从ffplay.c源码中粗略分析，我发现当前ffplay播放器的架构由4种类型的线程构成，所有的线程类型和其相应的功能描述如下：**

1. **SDL窗口主线程（main）：**该线程是ffplay程序的主线程，以main函数为入口，首先初始化了ffmpeg和SDL上下文，随后调用`stream_open`函数打开输入文件/流，并且完成相关的子线程创建逻辑。最后，进入SDL窗口的`event_loop`循环。在每次循环中，先通过`refresh_loop_wait_event`执行图像渲染逻辑，随后在`switch(event.type)`中处理SDL窗口消息，实现鼠标和键盘控制全屏、暂停播放、继续播放、退出等人机交互功能。
2. **解复用线程（read_thread）：**该线程由主线程调用的`stream_open`中的`is->read_tid   = SDL_CreateThread(read_thread, "read_thread", is);`一句创建，主要负责对于输入流/文件的解复用工作，即提取`AVPacket`然后缓存到对应的`PacketQueue`中。解复用线程启动后，首先通过ffmpeg提供的`avformat_open_input(...)`方法打开输入流/文件。随后，设置相关扫描参数后，调用`avformat_find_stream_info`确认输入流/文件是否含有有效的stream信息。紧接着，解复用线程通过`av_find_best_stream`方法，找到音频、视频和字幕对应的流索引，存放在`st_index[AVMEDIA_TYPE_NB]`数组中。随后，使用`stream_component_open`打开每个流，为每个流创建对应的解码线程（详述见下）。在上面的步骤都完成后，解复用线程进入自己的线程循环，每次循环中的主要工作就是通过`av_read_frame`从输入流/文件中读出一个`AVPacket`，然后根据这个`pkt`的`pkt->stream_index`，调用`packet_queue_put`把它缓存到每个stream对应的`PacketQueue`中，等待在`decoder_decode_frame`函数调用中被`if (packet_queue_get(d->queue, d->pkt, 1, &d->pkt_serial) < 0)`一句拿出然后送到解码器去解码。（`PacketQueue`结构体分析见下文）
3. **解码线程（video_thread, audio_thread, subtitle_thread)：**这一类线程负责对于`AVPacket`的解码工作，ffplay中解码线程有`video_thread`，`audio_thread`和`subtitle_thread`三个，这里字幕解码不做讨论。这些线程都是在解复用线程`read_thread`中，由`stream_component_open`过程中的`decoder_start`函数调用创建的。笼统来看，解码线程的主体都是一个`for(;;)`循环，每次循环中的流程首先是获取一个对应的`AVFrame`视频或者音频帧（`audio_thread`中直接调用`decoder_decode_frame`获取音频帧，而`video_thread`中调用`get_video_frame`获取视频帧，它是对于`decoder_decode_frame`的封装）。随后，根据解码后的`AVFrame`中的数据，计算相关参数。最后，将`AVFrame`添加到视音频对应的`FrameQueue`队列中（`audio_thread`中调用`frame_queue_push`，而`video_thread`中调用`queue_picture`，它是对于`frame_queue_push`的封装），分别供音频播放线程和主线程在音频播放和图像渲染时使用。（`FrameQueue`结构体分析见下文）
4. **音频播放线程（sdl_audio_callback）：**该线程由解复用线程在调用`stream_component_open`打开音频流的时候，`stream_component_open`内部调用`audio_open`函数，`audio_open`内部再调用`SDL_OpenAudioDevice`创建。该线程的主要工作逻辑实现在`sdl_audio_callback`中，该回调函数会在音频播放线程中被反复调用，来向外部请求可播放的音频数据。回调函数的类型声明语句是`typedef void (SDLCALL * SDL_AudioCallback) (void *userdata, Uint8 * stream, int len)`，在音频播放线程调用的时候，会填入`len`参数来告知需要给SDL送入多少字节的数据。随后，我们只需要将需要送入的数据拷贝到`stream`指向的缓冲区即可。这里ffplay在`sdl_audio_callback`回调函数实现中，首先通过`audio_decode_frame`从音频帧的`FrameQueue`中获取一帧音频数据（调用`frame_queue_peek_readable`实现），随后通过`swr_convert`对于音频数据进行重采样，处理后的音频数据放在`is->audio_buf`中返回`sdl_audio_callback`，由回调函数逻辑调用`memcpy`将数据复制到`stream`中送给SDL进行播放。

**综上所述，ffplay整体的架构图如下所示：**
![ffplay_arch](https://github.com/leo4048111/ffplay-explained/blob/3c16fac949b361b1a6fd24331ea47bbdb3866111/ffplay_arch.png)

# 核心结构体分析

## PacketQueue

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

## FrameQueue

`FrameQueue`结构体的声明如下所示：

```cpp
typedef struct FrameQueue
{
    Frame queue[FRAME_QUEUE_SIZE]; /* 用于存放帧数据的队列 */
    int rindex;                    /* 读索引 */
    int windex;                    /* 写索引 */
    int size;                      /* 队列中的帧数 */
    int max_size;                  /* 队列最大缓存的帧数 */
    int keep_last;                 /* 播放后是否在队列中保留上一帧不销毁 */
    int rindex_shown;              /* keep_last的实现，读的时候实际上读的是rindex + rindex_shown，分析见下 */
    SDL_mutex *mutex;              /* 互斥锁，用于保护队列操作 */
    SDL_cond *cond;                /* 条件变量，用于解码和播放线程的相互通知 */
    PacketQueue *pktq;             /* 指向对应的PacketQueue，FrameQueue里面的数据就是这个队列解码出来的 */
} FrameQueue;
```

可见这里`FrameQueue`中的`Frame`存储和`PacketQueue`不同，没有使用`AVFifo *`，而是通过一个循环数组`queue`来模拟队列，`rindex`指向当前读取的位置（即队头），`windex`指向当前写入的位置（即队尾），两者之间就是队列的数据范围。其中，`Frame`的数据都是从`pktq`指向的`PacketQueue`中解码得到的，`Frame`结构中就是`AVFrame`的数据外加一些其它的不同类型`Frame`参数，这样的设计应该是为了使得`Frame`结构能够同时存储视音频和字幕等不同类型的帧，这里不再赘述。下面结合对于`FrameQueue`结构体的操作，简要阐述这里面变量的作用：

+ `frame_queue_init`：该方法用来初始化一个`FrameQueue`，其中变量的初始化操作很容易理解，可以直接看源代码。注意`f->max_size = FFMIN(max_size, FRAME_QUEUE_SIZE);`一句，队列的理论最大大小是`FRAME_QUEUE_SIZE`即开辟的数组大小，但是实际上用的时候的`max_size`是可能小于`FRAME_QUEUE_SIZE`的，具体取决于初始化时传的参数。还有`f->keep_last = !!keep_last;`的写法本人第一次见，应该是因为`c`语言没有`bool`关键字，而`keep_last`在逻辑上又是一个布尔值，所以为了防止传进来非`0`和`1`的初始化数值，比如说`999`，用`!!`可以强制将这种数值转成`1`，这个`trick`学到了。

```cpp
static int frame_queue_init(FrameQueue *f, PacketQueue *pktq, int max_size, int keep_last)
{
    int i;
    memset(f, 0, sizeof(FrameQueue));
    if (!(f->mutex = SDL_CreateMutex()))
    {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateMutex(): %s\n", SDL_GetError());
        return AVERROR(ENOMEM);
    }
    if (!(f->cond = SDL_CreateCond()))
    {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateCond(): %s\n", SDL_GetError());
        return AVERROR(ENOMEM);
    }
    f->pktq = pktq;
    f->max_size = FFMIN(max_size, FRAME_QUEUE_SIZE);
    f->keep_last = !!keep_last;
    for (i = 0; i < f->max_size; i++)
        if (!(f->queue[i].frame = av_frame_alloc()))
            return AVERROR(ENOMEM);
    return 0;
}
```

+ `frame_queue_destory`：该函数用于销毁一个`FrameQueue`(这个名字比较奇怪，怀疑destory是拼错了，已经email提了MR，开发者反馈LGTM Thx，估计就是拼错了，不知道commit能不能合进去)。该函数的核心流程见源代码，做的工作比较简单，就是遍历了`f->max_size`范围下的所有`Frame`，然后解除内部对于`AVFrame`数据缓冲区的引用然后释放。

```cpp
static void frame_queue_destory(FrameQueue *f)
{
    int i;
    for (i = 0; i < f->max_size; i++)
    {
        Frame *vp = &f->queue[i];
        frame_queue_unref_item(vp);
        av_frame_free(&vp->frame);
    }
    SDL_DestroyMutex(f->mutex);
    SDL_DestroyCond(f->cond);
}
```

+ `frame_queue_signal`：内部就是对于`SDL_CondSignal(f->cond);`的封装，用于线程通信。

```cpp
static void frame_queue_signal(FrameQueue *f)
{
    SDL_LockMutex(f->mutex);
    SDL_CondSignal(f->cond);
    SDL_UnlockMutex(f->mutex);
}
```

+ `frame_queue_peek_writable`：这个函数用于返回一个队列中的可写位置。从`ffplay.c`源码来看，`FrameQueue`的写入操作一般分为三步。首先通过`frame_queue_peek_writable`获取一个可写位置。随后，直接用`=`对于位置中的`Frame`结构进行赋值。最后，再调用`frame_queue_push`来更新`FrameQueue`来更新`windex`写指针的位置。这里的这个函数逻辑其实很简单，就是返回了`windex`指向的可写位置，其它特殊情况的处理见源码

```cpp
static Frame *frame_queue_peek_writable(FrameQueue *f)
{
    /* wait until we have space to put a new frame */
    SDL_LockMutex(f->mutex);
    while (f->size >= f->max_size &&
           !f->pktq->abort_request)
    {
        SDL_CondWait(f->cond, f->mutex);
    }
    SDL_UnlockMutex(f->mutex);

    if (f->pktq->abort_request)
        return NULL;

    return &f->queue[f->windex];
}
```

+ `frame_queue_push`：上面提到了，这个函数的作用就是更新`windex`和`size`。并且，由于`FrameQueue`的队列实现是一个循环数组，因此如果`f->windex`加到了`f->max_size`，那么就回到0索引，起到一个循环的效果。

```cpp
static void frame_queue_push(FrameQueue *f)
{
    if (++f->windex == f->max_size)
        f->windex = 0;
    SDL_LockMutex(f->mutex);
    f->size++;
    SDL_CondSignal(f->cond);
    SDL_UnlockMutex(f->mutex);
}
```

+ `frame_queue_peek,frame_queue_peek_next,frame_queue_peek_last`：这三个`peek`函数返回的是分别是下一帧，下一帧的下一帧，和当前的队首。其中第一个函数可见返回元素的索引使用的是`return &f->queue[(f->rindex + f->rindex_shown) % f->max_size];`，这其实是`keep_last`实现逻辑的一部分，具体原理下面叙述。

```cpp
static Frame *frame_queue_peek(FrameQueue *f)
{
    return &f->queue[(f->rindex + f->rindex_shown) % f->max_size];
}

static Frame *frame_queue_peek_next(FrameQueue *f)
{
    return &f->queue[(f->rindex + f->rindex_shown + 1) % f->max_size];
}

static Frame *frame_queue_peek_last(FrameQueue *f)
{
    return &f->queue[f->rindex];
}
```

+ `frame_queue_peek_readable`：这个函数的作用是返回队列中的下一帧。这里可以看到返回的时候，使用的索引同样是`(f->rindex + f->rindex_shown) % f->max_size`。容易想象，如果`f->rindex_shown`是`0`，那么返回的就是队首。而如果`f->rindex_shown`是1，那么返回的是排在队首的后面一个元素。这样设计的用意是为了实现`keep_last`，即播放后保留上一次播放的帧。具体逻辑需要结合紧接着的`frame_queue_next`函数逻辑进行叙述。

```cpp
static Frame *frame_queue_peek_readable(FrameQueue *f)
{
    /* wait until we have a readable a new frame */
    SDL_LockMutex(f->mutex);
    while (f->size - f->rindex_shown <= 0 &&
           !f->pktq->abort_request)
    {
        SDL_CondWait(f->cond, f->mutex);
    }
    SDL_UnlockMutex(f->mutex);

    if (f->pktq->abort_request)
        return NULL;

    return &f->queue[(f->rindex + f->rindex_shown) % f->max_size];
}
```

+ `frame_queue_next`：这个函数的作用是推进队首指针`rindex`（即类似队列的pop操作）。这里注意第一句`if (f->keep_last && !f->rindex_shown) {...}`，其中的`keep_last`是在`init`的时候就写死的`1`或者`0`，表示这个队列是否需要保留播放完毕的上一帧。因为从`frame_queue_next`函数中可以看到，每次`rindex`自增前，都会对于之前已经被拿走的帧调用`frame_queue_unref_item`来解除引用。因此，ffplay这里在`keep_last`为1且`rindex_shown为0`，即第一次进入`frame_queue_next`这个函数的时候，会将`f->rindex_shown`置1，然后直接返回。这个操作使得第一次`frame_queue_next`之后`rindex`不变，和上次播放的那一帧的`index`一样，同时由于函数直接返回了，自然不会去释放第一帧。随后，在`frame_queue_peek_readable`的逻辑里面，我们可以看到，由于这个时候`rindex_show`已经恒为1了，所以每次返回的其实是队首的后一帧，取出的帧依然是新的下一帧。然后，每次进到`frame_queue_next`里面后，释放的其实就是上上帧，这样可以一直保证队列里面队首就是被`keep_last`保留的上一个播放完毕的帧，而队首的下一帧就是`frame_queue_peek_readable`取出的被送去播放的帧，这个设计与代码实现可谓妙哉妙哉，值得学习。

```cpp
static void frame_queue_next(FrameQueue *f)
{
    if (f->keep_last && !f->rindex_shown)
    {
        f->rindex_shown = 1;
        return;
    }
    frame_queue_unref_item(&f->queue[f->rindex]);
    if (++f->rindex == f->max_size)
        f->rindex = 0;
    SDL_LockMutex(f->mutex);
    f->size--;
    SDL_CondSignal(f->cond);
    SDL_UnlockMutex(f->mutex);
}
```

## VideoState

`VideoState`结构体声明如下：

```cpp
typedef struct VideoState
{
    SDL_Thread *read_tid;         /* 读线程 */
    const AVInputFormat *iformat; /* 输入文件格式 */
    int abort_request;            /* =1时请求退出播放 */
    int force_refresh;            /* =1时请求立即刷新画面 */
    int paused;                   /* =1时请求暂停播放，=0时播放 */
    int last_paused;              /* 暂存“暂停”和“播放”状态 */
    int queue_attachments_req;    /* =1时请求读取附加数据 */
    int seek_req;                 /* =1时请求seek */
    int seek_flags;               /* seek标志 */
    int64_t seek_pos;             /* 本次seek的目标位置（当前位置+增量） */
    int64_t seek_rel;             /* 本次seek的位置增量 */
    int read_pause_return;        /* 读线程暂停后的返回值 */
    AVFormatContext *ic;          /* iformat上下文 */
    int realtime;                 /* =1为实时播放 */

    Clock audclk; /* 音频时钟 */
    Clock vidclk; /* 视频时钟 */
    Clock extclk; /* 外部时钟 */

    FrameQueue pictq; /* 视频帧队列 */
    FrameQueue subpq; /* 字幕帧队列 */
    FrameQueue sampq; /* 音频帧队列 */

    Decoder auddec; /* 音频解码器 */
    Decoder viddec; /* 视频解码器 */
    Decoder subdec; /* 字幕解码器 */

    int audio_stream; /* 音频流索引 */

    int av_sync_type; /* 音视频同步类型（默认同步到音频时钟，即audio master） */

    double audio_clock;     /* 音频播放时钟（当前帧pts+duration） */
    int audio_clock_serial; /* 播放序列号，可被seek设置 */

    /* 下面4个参数在非audio master同步时使用 */
    double audio_diff_cum; /* used for AV difference average computation */
    double audio_diff_avg_coef;
    double audio_diff_threshold;
    int audio_diff_avg_count;

    AVStream *audio_st;                  /* 音频流 */
    PacketQueue audioq;                  /* 音频包队列 */
    int audio_hw_buf_size;               /* SDL音频缓冲区大小（单位B) */
    uint8_t *audio_buf;                  /* 指向待播放的一帧音频数据，在audio_decode_frame中被设置，如果重采样则指向重采样得到的音频数据，否则指向frame中的数据 */
    uint8_t *audio_buf1;                 /* 指向重采样得到的音频数据 */
    unsigned int audio_buf_size;         /* audio_buf指向缓冲区大小（单位B) */
    unsigned int audio_buf1_size;        /* audio_buf1指向缓冲区大小（单位B) */
    int audio_buf_index;                 /* 当前audio_buf中待拷贝数据的第一个字节的索引 */
    int audio_write_buf_size;            /* is->audio_buf_size - is->audio_buf_index，待拷贝字节数 */
    int audio_volume;                    /* 音量 */
    int muted;                           /* =1时静音 */
    struct AudioParams audio_src;        /* 音频源参数 */
    struct AudioParams audio_filter_src; /* 音频滤波器源参数 */
    struct AudioParams audio_tgt;        /* 音频目标参数 */
    struct SwrContext *swr_ctx;          /* 重采样上下文 */
    int frame_drops_early;               /* 丢弃的packet数 */
    int frame_drops_late;                /* 丢弃的frame数 */

    /* 显示模式（视频、波形...） */
    enum ShowMode
    {
        SHOW_MODE_NONE = -1,
        SHOW_MODE_VIDEO = 0,
        SHOW_MODE_WAVES,
        SHOW_MODE_RDFT,
        SHOW_MODE_NB
    } show_mode;
    int16_t sample_array[SAMPLE_ARRAY_SIZE];
    int sample_array_index;
    int last_i_start;
    RDFTContext *rdft;
    int rdft_bits;
    FFTSample *rdft_data;
    int xpos;
    double last_vis_time;
    SDL_Texture *vis_texture; /* 视频纹理 */
    SDL_Texture *sub_texture; /* 字幕纹理 */
    SDL_Texture *vid_texture; /* 视频纹理 */

    int subtitle_stream;   /* 字幕流索引 */
    AVStream *subtitle_st; /* 字幕流 */
    PacketQueue subtitleq; /* 字幕包队列 */

    double frame_timer;                 /* 最后一帧播放的时刻 */
    double frame_last_returned_time;    /* 最后一帧返回的时刻 */
    double frame_last_filter_delay;     /* 最后一帧滤波延迟 */
    int video_stream;                   /* 视频流索引 */
    AVStream *video_st;                 /* 视频流 */
    PacketQueue videoq;                 /* 视频包队列 */
    double max_frame_duration;          // maximum duration of a frame - above this, we consider the jump a timestamp discontinuity
    struct SwsContext *sub_convert_ctx; /* 字幕转换上下文 */
    int eof;

    char *filename;
    int width, height, xleft, ytop;
    int step;

    int vfilter_idx;
    AVFilterContext *in_video_filter;  // the first filter in the video chain
    AVFilterContext *out_video_filter; // the last filter in the video chain
    AVFilterContext *in_audio_filter;  // the first filter in the audio chain
    AVFilterContext *out_audio_filter; // the last filter in the audio chain
    AVFilterGraph *agraph;             // audio filter graph

    int last_video_stream, last_audio_stream, last_subtitle_stream; /* 最近的相关流索引 */

    SDL_cond *continue_read_thread; /* 当读取数据队列满了后进入休眠时，可以通过该condition唤醒该读线程 */
} VideoState;
```

`VideoState`是整个ffplay的核心管理者，所有资源的申请和释放以及线程的状态变化都是由其管理。从ffplay源码来看，这个数据结构的实例可以看做是一个单例，通过opaque指针在不同线程之间传递。虽然它是在main函数中的`stream_open`调用中通过`av_mallocz`创建，但是其中的变量会被各个线程使用和设置，因此其中的变量很多，功能也比较繁杂。涉及到比较关键的变量功能，将会在下文中针对性地进行阐述。

## Clock

`Clock`结构体声明如下：

```cpp
typedef struct Clock
{
    double pts;       /* clock base */
    double pts_drift; /* clock base minus time at which we updated the clock */
    double last_updated;
    double speed;
    int serial; /* clock is based on a packet with this serial */
    int paused;
    int *queue_serial; /* pointer to the current packet queue serial, used for obsolete clock detection */
} Clock;
```

这个结构体是`ffplay`中的时钟结构。`ffplay`中一共有三个时钟，分别是`audclk`，`vidclk`和`extclk`。时钟的主要功能是参与音视频同步的计算，具体原理下文中会详细阐述。

## Decoder

`Decoder`结构体声明如下：

```cpp
typedef struct Decoder
{
    AVPacket *pkt;
    PacketQueue *queue;
    AVCodecContext *avctx;
    int pkt_serial;
    int finished;
    int packet_pending;
    SDL_cond *empty_queue_cond;
    int64_t start_pts;
    AVRational start_pts_tb;
    int64_t next_pts;
    AVRational next_pts_tb;
    SDL_Thread *decoder_tid;
} Decoder;
```

该结构体主要是封装了`ffmpeg`中的`AVCodecContext*`解码器上下文，在这个基础上加上了一些其它参数，比如说取`Packet`的队列指针，解码器线程`tid`等等。同样，这里面的参数在用到的时候我再进行详细研究分析。

# 核心操作实现原理分析

## Start(开始播放)

ffplay的start过程基本上已经在上文中的架构图中能够比较清晰地呈现了，这里我再用一张图更加具体地给出ffplay的start过程中相关重要的函数调用逻辑和线程之间的数据通路，如下：

![ffplay_play](https://github.com/leo4048111/ffplay-explained/blob/a8857dcbee50bddf380f29e5c08880c026dc709d/ffplay_play.png)

流程上，大体就是从`main`的函数调用开始，先通过`stream_open`打开输入流/文件，然后初始化帧队列、包队列以及时钟结构，紧接着创建解复用线程后返回，进入`event_loop`。`event_loop`中如果没有`SDL`事件，代码就会一直运行在`refresh_loop_wait_event`的循环中，执行`video_refresh`的图像渲染逻辑。`video_refresh`中会根据`SHOW_MODE_VIDEO`是否被设置，决定是渲染波形图还是渲染视频。这里不考虑渲染波形图的情况，所以代码进入`retry:`下的逻辑。这里就是ffplay的音视频同步逻辑实现位置，具体算法待下文阐述。最后，音视频同步逻辑执行完毕后，进入`display:`下的逻辑，调用`video_display`把画面交给`SDL`渲染，然后循环往复。

在解复用线程中，主要工作就是创建音视频解码线程，然后往音视频解码线程读出`Packet`的`Packet`队列（is->videoq, is->audioq）里面放入待解码的`Packets`数据。音视频解码线程创建后，不停地从各自的`PacketQueue`中取包然后解码送进`FrameQueue`，即`is->pictq`和`is->sampq`中。第一个`FrameQueue`的数据会在上面`video_refresh`的代码中被访问取出使用，而第二个`FrameQueue`中的数据会在`sdl_audio_callback`回调函数的执行逻辑中被取出然后塞进`stream`缓冲区里面送到`SDL`进行播放。

## Pause(暂停播放)

ffplay的暂停播放逻辑入口在`SDL`事件处理的`event_loop`中，在用户按下`p`或者`space`的时候触发，进入函数`toggle_pause`，大致的函数调用流程和数据通路如下：

![ffplay_pause](https://github.com/leo4048111/ffplay-explained/blob/8ed9b7185c18c079e465142c98d802102307cc8f/ffplay_pause.png)

这里可以看到，按下按键后，首先进入到`toggle_pause`中，里面主要做2个工作。第一个是调用`stream_toggle_pause`来更新`is->paused`、外部时钟和三个时钟的`paused`状态。第二个是将`is->step`置为了0（这个变量用于实现播放器的按键`s`功能，即按一下往后走一帧，具体原理见下文对于`Step`功能的原理分析）。代码中，用到`is->paused`和三个时钟的`paused`变量的位置主要有4处。第一个地方是`refresh_loop_wait_event`对于`video_refresh`的调用位置，源码如下：

```cpp
static void refresh_loop_wait_event(VideoState *is, SDL_Event *event)
{
	...	
        if (is->show_mode != SHOW_MODE_NONE && (!is->paused || is->force_refresh))
            video_refresh(is, &remaining_time);
	...
}

static void video_refresh(void *opaque, double *remaining_time)
{
    ...
        if (is->paused)
            goto display;
    ...
}
```

这里可以看到，如果`is->paused`为1即暂停时，video_refresh只可能在`is->force_refresh`被设置的时候才会被调用。这个`is->force_refresh`在源码中可以看出，会在窗口大小更改的时候被设置。所以`video_refresh`中`is->paused`被设置时，直接可以跳过后面的音视频同步代码，直接进入`display`来渲染画面。

第二个用到`audio_decode_frame`里面，源代码如下：

```cpp
static int audio_decode_frame(VideoState *is)
{
	...
    if (is->paused)
        return -1;
    ...   
}
```

这里的目的是为了在暂停时不从`is->sampq`中取数据，实现暂停时的静音效果。因为从ffplay的源码来看，解码线程是没有暂停这个操作的，就算`is->paused`被设置了，播放器的播放暂停，但是解码工作还是会继续，直到把缓存队列塞满或者无码可解。所以，这里让`audio_decode_frame`直接返回-1，不去从`FrameQueue`中读数据，这样外边调用它的`sdl_audio_callback`看到返回值`audio_size < 0`，就会把`is->audio_buf`设为`NULL`，然后后面送数据的时候就会走到`memset(stream, 0, len1)`的逻辑，从而往`SDL`里面送空数据，实现了暂停时的音频也一样暂停。

第三个用到`is->paused`的地方是解复用线程`read_thread`，这个里面的代码逻辑如下：

```cpp
static int read_thread(void *arg) {
...
        if (is->paused != is->last_paused)
        {
            is->last_paused = is->paused;
            if (is->paused)
                is->read_pause_return = av_read_pause(ic);
            else
                av_read_play(ic);
        }
...
}
```

可以看到，这里做了一个判断，如果暂停的状态发生了改变（比如原来在播放现在暂停，或者原来暂停现在播放），就分别调用`av_read_pause`和`av_read_play`来暂停或者继续从流/文件中读取`AVPacket`。这三处代码块通过访问`is->paused`变量的状态，执行不同的代码逻辑，从而实现了播放器的暂停功能。

最后，可以发现在暂停的时候，三个`Clock`的`paused`变量也被设置为了`is->paused`的状态。这里设置的`paused`会在`get_clock`函数调用中被使用。这里`get_clock`的计算原理暂时不进行阐述，留到视音频同步算法解析部分一并分析研究。

## Step(逐帧前进)

在看上面ffplay暂停功能的源码实现时，我发现了在暂停时还设置了`is->step`这个变量等于0，不知道意义何在，于是简单研究了一下它的用途。最终，我定位到了`event_loop`里面的`step_to_next_frame`代码，简单看了一下逻辑，发现这是一个逐帧前进的功能实现。在`ffplay`中，对于正在播放的视频，按一下`s`键，就会往后前进一帧并且暂停，随后每按一下`s`前进一帧，视频保持暂停。这个功能实现的源码如下：

```cpp
static void step_to_next_frame(VideoState *is)
{
    /* if the stream is paused unpause it, then step */
    if (is->paused)
        stream_toggle_pause(is);
    is->step = 1;
}

static void video_refresh(void *opaque, double *remaining_time)
{
    ...
    if (is->step && !is->paused)
        stream_toggle_pause(is);
    ...
}
```

这里可以看到，`step_to_next_frame`中第一步是如果视频正在暂停，就让视频继续播放。随后，将`is->step`设置为1，表示现在正在进行逐帧前进操作。随后，在下一次进入`video_refresh`的时候，会走到上面的这个代码里面。这时候视频肯定是正在播放的状态，所以里面调用了`stream_toggle_pause`就把视频暂停了，这时候紧接着代码执行当前读出的这一帧的渲染逻辑，这一帧渲染完后视频就是一个暂停的状态。然后如果再按一下`s`，视频又往后渲染一帧之后暂停，以此类推，直到用户按`space`或者`p`，调用`stream_toggle_pause`的时候清除`is->step`，才能让视频继续播放。所以可以看出，逐帧前进这个功能的实现其实就是通过按一下按键，播放器就播放一帧后暂停，再按一下就再播放一帧后暂停这样的逻辑实现的，这是`is->step`变量设置的意义所在。

## Seek(跳转播放)

Seek操作的主要流程大致为下图所示：

![ffplay_seek](https://github.com/leo4048111/ffplay-explained/blob/65cd7bc99f705d41dc95a827ff4c164dd135ff55/ffplay_seek.png)

可见其中主要有三大步骤，第一是在`event_loop`中根据用户按的按钮不同，分别设置跳转的时长（目前看下来ffplay只支持快进快退10s或者60s。然后，根据文件类型不同，选择不同的`seek`方式。如果是按字节`seek`（`seek_by_bytes`），则通过`incr`乘上字节率（比特率 / 8）计算出需要seek偏移（这里单位字节）。否则，直接在计算中用`incr`和当前`get_master_clock`返回的主时钟相加即可，这时候算出来的`pos`和`rel`单位是秒。

上一步执行完毕后，第二步两个分支都会调用`stream_seek`将计算完毕的`pos`和`rel`设置到`is`的`seek`参数中（其中上面算出以秒为单位的数据在传进来的时候转成了微秒）。`stream_seek`函数只进行参数设置，然后把`seek_req`置1，表示当前请求`seek`操作，本身它并不做实际的seek工作。

第三步之前的上述工作由主线程完成，最后在`is->seek_req`被设置后，解复用线程`read_thread`会进入到下面的代码逻辑中：

```cpp
if (is->seek_req)
        {
            int64_t seek_target = is->seek_pos;
            int64_t seek_min = is->seek_rel > 0 ? seek_target - is->seek_rel + 2 : INT64_MIN;
            int64_t seek_max = is->seek_rel < 0 ? seek_target - is->seek_rel - 2 : INT64_MAX;
            // FIXME the +-2 is due to rounding being not done in the correct direction in generation
            //      of the seek_pos/seek_rel variables

            ret = avformat_seek_file(is->ic, -1, seek_min, seek_target, seek_max, is->seek_flags);
            if (ret < 0)
            {
                av_log(NULL, AV_LOG_ERROR,
                       "%s: error while seeking\n", is->ic->url);
            }
            else
            {
                if (is->audio_stream >= 0)
                    packet_queue_flush(&is->audioq);
                if (is->subtitle_stream >= 0)
                    packet_queue_flush(&is->subtitleq);
                if (is->video_stream >= 0)
                    packet_queue_flush(&is->videoq);
                if (is->seek_flags & AVSEEK_FLAG_BYTE)
                {
                    set_clock(&is->extclk, NAN, 0);
                }
                else
                {
                    set_clock(&is->extclk, seek_target / (double)AV_TIME_BASE, 0);
                }
            }
            is->seek_req = 0;
            is->queue_attachments_req = 1;
            is->eof = 0;
            if (is->paused)
                step_to_next_frame(is);
        }
```

这里是真正实现seek操作的位置，原理是调用了`avformat_seek_file`这个函数。调用完之后，文件或者流的读取位置就被正确更新了，之后的`av_read_frame`都会从新的位置开始取出`AVPacket`。随后，ffplay会通过`packet_queue_flush`把`PacketQueue`缓存清空，同时重设外部时钟到seek到的新位置，然后清除`seek_req`的标志。最后，如果当前视频是暂停的状态，则进行一次step操作，目的是为了让播放器上显示seek到的最新位置的画面，最终实现了整体的seek操作逻辑。

# 视音频同步算法原理与代码实现分析

**这里所涵盖的函数与算法分析主要围绕音视频同步算法中所用到的相关变量设置与函数调用流程开展，先介绍例如`audclk`,`vidclk`,`audio_clock`等涉及时间计算的变量的设置时机与计算方式，最后分析ffplay所使用的视音频同步算法原理及其实现。**

## audio_clock变量的设置时机与计算方法

在`VideoState`里面有两个和音频时钟计算有关的变量。一个是`audclk`，它是一个`Clock`数据结构对象。另一个是一个`double`型数据`audio_clock`。这里我先分析一下`audio_clock`变量的含义、功能、设置时机与计算方式。这个变量的设置是在`audio_decode_frame`中进行的，代码如下：

```cpp
static int audio_decode_frame(VideoState *is)
{
...
/* update the audio clock with the pts */
    if (!isnan(af->pts))
        is->audio_clock = af->pts + (double)af->frame->nb_samples / af->frame->sample_rate;
    else
        is->audio_clock = NAN;
...
}
```

这里`audio_decode_frame`这个函数在每次`sdl_audio_callback`时都会被调用，用来从`FrameQueue`中取出一帧解码后的音频数据，进行一定的加工（比如说重采样和采样率调整、通道数调整等，一般不会被执行），最后将其中的数据返回给`sdl_audio_callback`。在最后，可以看到`audio_clock`的计算是当前取出这一帧的`pts`加上了`(double)af->frame->nb_samples / af->frame->sample_rate`。这里的这个`(double)af->frame->nb_samples / af->frame->sample_rate`很明显就是帧的`duration`时长，因此我们可以得出结论，`is->audio_clock`应该是等于当前最新被`audio_decode_frame`从`FrameQueue`取出的帧的`pts` + `duration`，也就是这帧被播放完时刻的时间戳。至于这里设置完`audio_clock`和`is->audclk`的音频时钟计算有什么关系，我们下文继续阐述。

## audclk变量的设置时机与计算方法

`audclk`变量的设置时机紧接着上面所述的`audio_clock`变量，在`sdl_audio_callback`调用`audio_decode_frame`并且返回后，`sdl_audio_callback`会使用`memcpy`将`Frame`中的数据拷进`SDL`，如果不够拷那就再取下一个`frame`放到`is->audio_buff`里面，直到`len == 0`为止，同时更新`is->audio_buf_index`和`is->audio_write_buf_size`。紧接着，关键的计算代码如下：

```cpp
static void sdl_audio_callback(void *opaque, Uint8 *stream, int len)
{
...
    /* Let's assume the audio driver that is used by SDL has two periods. */
    if (!isnan(is->audio_clock))
    {
        set_clock_at(&is->audclk, is->audio_clock - (double)(2 * is->audio_hw_buf_size + is->audio_write_buf_size) / is->audio_tgt.bytes_per_sec, is->audio_clock_serial, audio_callback_time / 1000000.0);
        sync_clock_to_slave(&is->extclk, &is->audclk);
    }
...
}
```

这里可以看到，`set_clock_at`函数调用时传入的`pts`参数是`is->audio_clock - (double)(2 * is->audio_hw_buf_size + is->audio_write_buf_size) / is->audio_tgt.bytes_per_sec`。参考CSDN博客文章[FFplay源码分析-音视频同步1](https://blog.csdn.net/u012117034/article/details/122873602?spm=1001.2014.3001.5506)的分析内容，后面的这个`2 * is->audio_hw_buf_size + is->audio_write_buf_size`这个数值的含义其实就是当前还没有开始播的被缓存数据字节数。这个算式由两部分构成，首先是前面的`2 * is->audio_hw_buf_size`，这个是在`SDL`内部的未播完数据长度，结构如下所示：
![image](https://github.com/leo4048111/ffplay-explained/assets/74029782/0771e0ad-4fb6-468c-9aa6-194bbf9df1ca)

这里可见一段红色的，就是SDL里面正在播的数据，长度为`is->audio_hw_buf_size`，后面那个`len`就是这次`sdl_audio_callback`函数调用中我们手动拷进去的数据长度。这里注意，SDL调用`sdl_audio_callback`拿数据的时候，传进来的`len`和`audio_hw_buf_size`是恒相等的。所以，在这里的计算中，`2 * is->audio_hw_buf_size`其实是`len + is->audio_hw_buf_size`。只不过`len`变量在上面的`while`循环取数据中已经被减成0了，所以这里直接就`2 * is->audio_hw_buf_size`进行计算。然后，算式的第二部分是`is->audio_write_buf_size`，这是`is->audio_buf`中还没有被送给`SDL`的剩余帧数据长度。拿这三部分的和除以`is->audio_tgt.bytes_per_sec`，算出的就是播完这三部分所用的时间。最后，用之前算出来的播完这一帧的时间戳`is->audio_clock`，减去播完这三段的总时间，得到的就是当前音频时钟`audclk`的准确`pts`，大概的图示如下：

![ffplay_audclk](https://github.com/leo4048111/ffplay-explained/blob/0c0290e74218de37ee1bd57631283fa1544dd48f/ffplay_audclk.png)

在确定了时钟pts后，通过`set_clock_at`设置`is->audclk`的`pts`，然后将`is->extclk`外部时钟同步到`is->audclk`上，以供在音视频同步计算中使用。

## 音视频同步算法解析

上面我们已经知道了如何正确计算音频时钟`audclk`，这时候就已经可以开始执行音视频同步算法流程了。这个算法通过计算每一帧的延迟`remaining_time`来控制`av_usleep`的睡眠时间，从而让帧和音频的播放保持在一个可接受的不同步范围内。如果帧落后音频太多，ffplay的音视频同步算法还会进行丢帧操作，来让视频播放快速追上音频。这个同步操作的代码实现据我观察主要分散在两个位置，分别由视频解码线程和主线程执行。而我们要重点阐述的核心算法位于第二个位置，处理逻辑由主线程进行执行，代码在`video_refresh`函数下进行实现。

首先，第一个涉及音视频同步代码逻辑的位置如下所示：

```cpp
static int get_video_frame(VideoState *is, AVFrame *frame)
{
...
	    if (got_picture)
    {
        double dpts = NAN;

        if (frame->pts != AV_NOPTS_VALUE)
            dpts = av_q2d(is->video_st->time_base) * frame->pts;

        frame->sample_aspect_ratio = av_guess_sample_aspect_ratio(is->ic, is->video_st, frame);

        if (framedrop > 0 || (framedrop && get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER))
        {
            if (frame->pts != AV_NOPTS_VALUE)
            {
                double diff = dpts - get_master_clock(is);
                if (!isnan(diff) && fabs(diff) < AV_NOSYNC_THRESHOLD &&
                    diff - is->frame_last_filter_delay < 0 &&
                    is->viddec.pkt_serial == is->vidclk.serial &&
                    is->videoq.nb_packets)
                {
                    is->frame_drops_early++;
                    av_frame_unref(frame);
                    got_picture = 0;
                }
            }
        }
    } 
...
}
```

这里上面已经提到了，视频解码线程获取解码后数据的方式是调用`get_video_frame`，这个函数是对于`decoder_decode_frame`即实际调用`avcodec_receive_frame`获取解码后数据的函数的一个封装。至于为什么音频解码直接调用的就是`decoder_decode_frame`获取解码后数据，而视频解码则需要这一个封装流程，原因就在上面我贴出的这段代码。因为音频解码后的数据不需要丢弃，直接往SDL里送就可以了，但是视频解码出来之后，这里先一步做了一个丢帧的操作，所以需要一层封装来wrap这个函数。从代码里来看，这里首先算了一个解码出来帧`dpts`，也就是`pts * timebase`，单位是秒。然后，如果同步模式不是`AV_SYNC_VIDEO_MASTER`，即不是以视频时钟为主时钟同步到视频（一般都不会是这种模式）的话，进入丢帧计算逻辑。先算出当前拿出来帧`dpts`和主时钟之间的差值`diff`。随后，如果这个差值在可同步阈值范围`AV_NOSYNC_THRESHOLD`内（这个阈值的作用就是控制如果差的太大就直接不走同步逻辑了），并且`diff - is->frame_last_filter_delay < 0`也就是`diff < 0`（这里的`is->frame_last_filter_delay`是个常数0），表示当前主时钟已经比拿出来的帧的`dpts`快了，也就是帧慢了。同时，在这个基础上，如果帧队列里面还有数据，就正式进入丢帧逻辑，`is->frame_drops_early++`记录丢掉的总帧数后，直接`av_frame_unref(frame);`丢帧，然后将`got_picture`设置为0，表示重新从队列取帧，直到帧的`dpts`追上甚至超过主时钟为止。所以，其实ffplay在解码的时候就已经开始进行初步的音视频同步操作了。

其次，第二个涉及音视频同步的位置，就是ffplay的音视频同步核心算法实现，在`video_refresh`函数中，实现主体如下：

```cpp
/* called to display each frame */
static void video_refresh(void *opaque, double *remaining_time)
{
	...
	    if (is->video_st) {
retry:
        if (frame_queue_nb_remaining(&is->pictq) == 0) {
            // nothing to do, no picture to display in the queue
        } else {
            double last_duration, duration, delay;
            Frame *vp, *lastvp;

            /* dequeue the picture */
            lastvp = frame_queue_peek_last(&is->pictq);
            vp = frame_queue_peek(&is->pictq);
            
			...
			
            /* compute nominal last_duration */
            last_duration = vp_duration(is, lastvp, vp);
            delay = compute_target_delay(last_duration, is);

            time= av_gettime_relative()/1000000.0;
            if (time < is->frame_timer + delay) {
                *remaining_time = FFMIN(is->frame_timer + delay - time, *remaining_time);
                goto display;
            }

            is->frame_timer += delay;
            if (delay > 0 && time - is->frame_timer > AV_SYNC_THRESHOLD_MAX)
                is->frame_timer = time;

            SDL_LockMutex(is->pictq.mutex);
            if (!isnan(vp->pts))
                update_video_pts(is, vp->pts, vp->serial);
            SDL_UnlockMutex(is->pictq.mutex);

            if (frame_queue_nb_remaining(&is->pictq) > 1) {
                Frame *nextvp = frame_queue_peek_next(&is->pictq);
                duration = vp_duration(is, vp, nextvp);
                if(!is->step && (framedrop>0 || (framedrop && get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) && time > is->frame_timer + duration){
                    is->frame_drops_late++;
                    frame_queue_next(&is->pictq);
                    goto retry;
                }
            }
}
```

其中两个重要函数调用分别是`vp_duration`和`compute_target_delay`，实现原理下文中将会详细阐述。整体来看，可以看到这个音视频同步的算法的基本流程和逻辑还是非常简洁明了的，下面我们结合其代码实现进行逐行分析：

+ **变量声明**：上面的代码中，我已经去掉了涉及seek后同步的相关代码逻辑，以便于进行分析。首先，可以看到算法入口处声明了5个变量，其中两个`Frame* vp, *lastvp`指向当前帧和上一帧（因为视音频队列初始化的时候都设置了`keep_last`，所以能够获取到上一帧，这也是`keep_last`的意义所在）。还有3个`double`型，`last_duration`表示上一帧的长度，`duration`表示当前帧的长度，`delay`是这个算法需要输出的延迟时间，用于最后根据这个`delay`进一步算出`remaining_time`，然后调整线程入睡时间，从而控制当前帧在SDL窗口中的呈现时长，对应的源码如下：

```cpp
...
double last_duration, duration, delay;
Frame *vp, *lastvp;
...
```

+ **获取上一帧和当前帧：**就是调用上面的`peek_last`和`peek`函数，实现原理上文已经阐述过了，见`FrameQueue`结构体分析，这里不再赘述，直接给出对应源码：

```cpp
...
/* dequeue the picture */
lastvp = frame_queue_peek_last(&is->pictq);
vp = frame_queue_peek(&is->pictq);
...
```

+ **计算`last_duration`：**这里开始，视音频同步算法正式进入计算环节。首先，这里计算了一个上一帧的`duration`。在`serial`相同即没有进行过`seek`的情况下，`vp->serial == nextvp->serial`肯定是`true`，所以进到里边的计算逻辑。这个计算也非常直观，就是拿新帧的`pts`减掉老帧的`pts`，获取一个准确的`duration`。然后如果这个`duration`算出来数据有问题（比如小于0等等），就直接用`vp`结构体里面的`vp->duration`作为结果。至于为什么不直接用`vp->duration`作为返回值，而是要算一遍`nextvp->pts - vp->pts`，个人认为可能用`pts`的这个数值的计算更加准确吧。这里也给出源码如下：

```cpp
...
/* compute nominal last_duration */
last_duration = vp_duration(is, lastvp, vp);
...

static double vp_duration(VideoState *is, Frame *vp, Frame *nextvp) {
    if (vp->serial == nextvp->serial) {
        double duration = nextvp->pts - vp->pts;
        if (isnan(duration) || duration <= 0 || duration > is->max_frame_duration)
            return vp->duration;
        else
            return duration;
    } else {
        return 0.0;
    }
}
```

+ **计算`delay`：**上面算完`last_duration`之后，紧接着再算一个`delay`。可以看到，`delay`的计算步骤稍微多一点。从计算实现代码`compute_target_delay`头部开始看，首先通过变量声明引入了两个概念`sync_threshold`和`diff`，需要在这里明确一下。先说第二个`diff`，在默认情况下，一般是把视频同步到音频时钟上，那么这里的`diff`就是当前视频时钟和主时钟（也就是音频时钟）的差值。而第一个`sync_threshold`表示同步阈值，如果`diff`差值在$[-sync\_threshold, +sync\_threshold]$这个范围内，认为这时候不需要同步，因为人眼基本看不出视音频之间的不同步情况。而$diff \leq -sync\_threshold$，就说明视频太慢了，这时候就要减小`delay`甚至是丢帧来让视频赶上音频。反之$diff \geq sync\_threshold$就说明视频放的太快了，这时候就要让当前帧播放的久一点（也就是主线程睡得久一点），来让音频赶上视频。

  建立了上面对于音视频同步原理的基本认知后，下面的代码就非常容易理解了。首先就是计算`diff = get_clock(&is->vidclk) - get_master_clock(is);`，然后计算一个`sync_threshold`（这里的同步阈值可以看到不是写死的，是算出来的，根据算式来看如果`delay`在$[AV\_SYNC\_THRESHOLD\_MIN, AV\_SYNC\_THRESHOLD\_MAX]$区间就是`delay`，否则就是区间两个端点）。计算完毕后，如果`diff`在可以处理的同步范围内（ffplay的逻辑是如果音视频失同步的差值太大就直接不同步了，随便放），那么分三种情况讨论。第一种`if (diff <= -sync_threshold)`就是视频太慢了，那就和上面说的一样，用`delay = FFMAX(0, delay + diff);`让`delay`变小（这个算式一般来说算出的`delay`就是0，因为`diff <= -sync_threshold`然后`sync_threshold`一般又等于`delay`）。然后第二和第三种情况都是视频比音频快，这时候根据快多少分别让`delay = delay + diff;`或者直接`delay = 2 * delay;`，很好理解，不再赘述了，算完了之后这个函数就返回了。

```cpp
...
delay = compute_target_delay(last_duration, is);
...

static double compute_target_delay(double delay, VideoState *is)
{
    double sync_threshold, diff = 0;

    /* update delay to follow master synchronisation source */
    if (get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER) {
        /* if video is slave, we try to correct big delays by
           duplicating or deleting a frame */
        diff = get_clock(&is->vidclk) - get_master_clock(is);

        /* skip or repeat frame. We take into account the
           delay to compute the threshold. I still don't know
           if it is the best guess */
        sync_threshold = FFMAX(AV_SYNC_THRESHOLD_MIN, FFMIN(AV_SYNC_THRESHOLD_MAX, delay));
        if (!isnan(diff) && fabs(diff) < is->max_frame_duration) {
            if (diff <= -sync_threshold)
                delay = FFMAX(0, delay + diff);
            else if (diff >= sync_threshold && delay > AV_SYNC_FRAMEDUP_THRESHOLD)
                delay = delay + diff;
            else if (diff >= sync_threshold)
                delay = 2 * delay;
        }
    }

    av_log(NULL, AV_LOG_TRACE, "video: delay=%0.3f A-V=%f\n",
            delay, -diff);

    return delay;
}
```

+ **计算`remaining_time`后直接返回`event_loop`：**上面两个关键数据变量计算完毕后，可以开始计算最终的`remaining_time`也就是线程要睡的时间，进入渲染或者丢帧逻辑了。这部分代码的第一个分支位于`if (time < is->frame_timer + delay)`。这里的`time`是当前时刻，`is->frame_timer`是上一帧被渲染的时刻，`delay`就是刚才算出来的这帧应该显示多久的数据。`time < is->frame_timer + delay`说明当前时间段这个帧应该在`SDL`窗口中上场渲染了。这时候直接开始计算`remaining_time`。为啥这里`remaining_time`不直接等于`delay`呢？这是因为上面的各种计算代码和上一帧渲染之后走的所有代码逻辑也会耗费时间，而这段时间就是算在`delay`里面的，所以说这时候要是直接睡`delay`就太多了。建立了这个认识之后，`*remaining_time = FFMIN(is->frame_timer + delay - time, *remaining_time);`这句也非常容易理解了，就是从`delay`中把之前代码的运行时间给扣掉，然后这时候算出来的`remaining_time`就能正确被外部`event_loop`拿去当做睡的时长了。

```cpp
time= av_gettime_relative()/1000000.0;
if (time < is->frame_timer + delay) {
    *remaining_time = FFMIN(is->frame_timer + delay - time, *remaining_time);
    goto display;
}
```

+ **如果上面没返回，那就开始丢帧：**这里就是第二个情况，可知当前的帧按照音视频同步逻辑显示完后的时间还比当前时刻早（没赶上当前时刻），就说明这个帧已经没法同步了，就得丢掉。丢的时候，首先更新`is->frame_timer += delay;`记录一下这个帧处理完的时刻，然后用丢掉的这个帧更新视频时钟（就是`update_video_pts(is, vp->pts, vp->serial);`，因为丢帧其实可以理解为这个帧已经放完了，所以这时候视频时钟的`pts`应该同步到这个帧的`pts`）。然后如果视频`FrameQueue`也就是`is->pictq`中还有帧，那么就更新一下`duration`，然后直接取下一帧，同时丢帧总数`is->frame_drops_late++;`更新一下，重新`retry`再走一遍上面的步骤，直到取出的帧能播为止。

```cpp
is->frame_timer += delay;
if (delay > 0 && time - is->frame_timer > AV_SYNC_THRESHOLD_MAX)
    is->frame_timer = time;

SDL_LockMutex(is->pictq.mutex);
if (!isnan(vp->pts))
    update_video_pts(is, vp->pts, vp->serial);
SDL_UnlockMutex(is->pictq.mutex);

if (frame_queue_nb_remaining(&is->pictq) > 1) {
    Frame *nextvp = frame_queue_peek_next(&is->pictq);
    duration = vp_duration(is, vp, nextvp);
    if(!is->step && (framedrop>0 || (framedrop && get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) && time > is->frame_timer + duration){
        is->frame_drops_late++;
        frame_queue_next(&is->pictq);
        goto retry;
    }
}
```

上面的逻辑走完后，算出来的`remaining_time`送回到`event_loop`，里面紧接着就是一个`if (remaining_time > 0.0) av_usleep((int64_t)(remaining_time * 1000000.0));`入睡，从而完美收尾了整套音视频同步逻辑。
