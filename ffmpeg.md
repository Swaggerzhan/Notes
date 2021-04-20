# ffmpeg

### 1. 函数部分

 *  avformat_open_input()
FFMPEG打开媒体的过程开始于avformat_open_input这个函数，它使得AVIOContext初始化，输入输出协议识别。
![](./ffmpeg_pic/2.jpg)

```C
int avformat_open_input(
    AVFormatContext **ps, // 函数调用成功后处理过的AVFormatContext结构体
    const char *filename, // 打开的音视频的URL
    AVInputFormat* fmt, // AVFormatContext中的AVInputFormat,为NULL自动检测
    AVDictionary **options, // 附加选项，一般为NULL
);

/* 成功返回值>=0 */
```

* avformat_alloc_context()
AVFormatContext 初始化函数

```C
AVFormatContext *avformat_alloc_context(void); // 初始化
void avformat_free_context(AVFormatContext* s); // 销毁
```

* avio_alloc_context()
AVIOContext 初始化函数

```C
AVIOContext *avio_alloc_context(
    unsigned char *buffer, // buffer
    int buffer_size, // buffer size
    int write_flag, // 1表示buffer可写，否则buffer只读
    void *opaque, // 用户自定义数据
    int (*read_packet)(void *opaque, uint8_t *buf, int buf_size),
    int (*write_packet)(void *opaque, uint8_t *buf, int buf_size),
    int64_t (*seek)(void *opaque, int64_t offset, int whence)
);
// 初始化
av_free(AVIOContext* s); // 销毁
```

read_packet()函数是一个读取外部数据填充buffer的函数。
write_packet()函数是一个往buffer中写入数据的函数。

* avformat_new_stream()
AVStream 初始化函数

```C
AVStream *avformat_new_stream(
    AVFormatContext *s,
    const AVCodec *c
); // 初始化
void avformat_free_context(); //销毁
```

* avcodec_alloc_context3()
AVCodecContext 初始化，一般是由AVStream初始化时候顺带初始化了。

```C
AVCodecContext *avcodec_alloc_context3(const AVCodec* codec);
```

* av_format_alloc()
AVFrame 结构体初始化

```C
AVFrame *av_frame_alloc(void); // 初始化
void av_frame_free(AVFrame **frame); // 销毁
```

* av_init_packet()
AVPacket 结构体初始化

```C
void av_init_packet(AVPacket *pkt); // 初始化 可能已经弃坑
static void get_packet_defaults(AVPacket *pkt);// 新初始化函数
int av_new_packet(AVPacket *pkt, int size); // 另外的一个初始化函数，内部调用了av_init_packet函数
void av_free_packet(AVPacket *pkt); // 销毁
```

* avio_open2()
该函数用于打开ffmpeg的输入和输出文件

```C
int avio_open2(
    AVIOContext **s, // 函数调用成功后创建的AVIOContext结构体
    const char *url, // 输入输出协议的地址(文件就是路径)
    int flags, // 标志位
    const AVIOIterruptCB *int_cb, // NULL
    AVDictionary **options // NULL
);
```
其中flags有3个选择
`AVIO_FLAG_READ` 只读
`AVIO_FLAG_WRITE` 只写
`AVIO_FLAG_READ_WRITE` 读写

* avcodec_find_encoder()以及avcodec_find_decoder()
函数用于寻找一个解码或者编码器

```C
AVCodec *avcodec_find_encoder( enum AVCodecID id);
AVCodec *avcodec_find_decoder( enum AVCodecID id);
// 没有找到返回NULL
```

* avcodec_open2()
函数用于初始化一个音视频编码器的AVCodecContext

```C
int avcodec_open2(
    AVCodecContext *avctx, // 需要初始化的AVCodecContext
    const AVCodec *codec, // 输入的AVCodec
    AVDictionary **options // 选项
);

int avcodec_close(AVCodecContext *avctx); // 关闭
```



### 2. 结构体

所有结构体大致是这种关系
![](./ffmpeg_pic/1.png)

`AVFormatContext`:统领全局的基本结构体。主要用于封装格式(FLV/MKV/RMVB等)。
`AVIOContext`: 输入输出对应结构体，用于输入输出(读写文件，RTMP协议等)。
`AVStream, AVCodecContext`: 音频流对应的结构体，用于视频编码。
`AVFrame`: 储存非压缩的数据(视频对应RGB/YUV像素数据，音频对应PCM采样数据)。
`AVPacket`: 储存压缩数据(视频对应H.264等码流数据，音频对应AAC/MP3等码流数据)。





引用: https://blog.csdn.net/leixiaohua1020/article/details/44064715