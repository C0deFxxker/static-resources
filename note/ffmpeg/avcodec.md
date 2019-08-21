# 1 视频基础概念
## 帧率、码率、比特率
* **帧率**: 视频每秒播放图片数。  
* **码率（比特率）**: 每秒显示的图片进行压缩后的数据位数。  
* **分辨率**: 每帧图片的宽度和高度。

他们之间有如下关系:  
帧率 * 分辨率 = 压缩前每秒处理的数据位数  
压缩比 = 压缩前每秒处理的数据位数 / 码率  
文件大小(字节) = 码率 / 8 * 时长(秒)

## I帧、P帧、B帧
* **I帧**: I帧又称帧内编码帧，是一种自带全部信息的独立帧，可以简单理解为一张静态画面。  
* **P帧**: P帧又称帧间预测编码帧，需要参考前面的I帧才能进行编码。表示的是当前帧画面与前一帧（前一帧可能是I帧也可能是P帧）的差别。P帧通常占用更少的数据位，但P帧依赖于前面帧的数据推算出来，如果前面的帧因网络传输异常出现偏差，当前P帧也会受牵连。  
* **B帧**: B帧又称双向预测编码帧，也就是B帧记录的是本帧与前后帧的差别。B帧不仅要取得之前的缓存画面，还要解码之后的画面，通过前后画面的与本帧数据的叠加取得最终的画面。B帧压缩率高，但是对解码性能要求较高。  
* **GOP**:  Group of Picture，关键帧的周期，也就是两个I帧之间的距离,一个帧组的最大帧数。在一个GOP中，P、B帧是由I帧预测得到的，当I帧的图像质量比较差时，会影响到一个GOP中后续P、B帧的图像质量；相反，若GOP太小会增加网络传输成本。
  
I帧只需考虑本帧；P帧记录的是与前一帧的差别；B帧记录的是前一帧及后一帧的差别,能节约更多的空间,视频文件小了,但相对来说解码效率更慢。视频监控系统中预览的视频画面是实时的，对画面的流畅性要求较高。采用I帧、P帧进行视频传输可以提高网络的适应能力，且能降低解码成本所以现阶段的视频解码都只采用I帧和P帧进行传输。海康摄像机编码，I帧间隔是50，含49个P帧（GOP=50）。

## 码率控制算法
动态调整编码器参数，得到目标比特数。为视频序列中的图像组GOP、图像或子图像分配一定的比特。现有的码率控制算法主要是通过调整离散余弦变换的量化参数（QP）大小输出目标码率。

## 编码模式
* **VBR**: Variable BitRate，动态比特率，其码率可以随着图像的复杂程度的不同而变化，因此其编码效率比较高，Motion发生时，马赛克很少。码率控制算法根据图像内容确定使用的比特率，图像内容比较简单则分配较少的码率(似乎码字更合适)，图像内容复杂则分配较多的码字，这样既保证了质量，又兼顾带宽限制。这种算法优先考虑图像质量。  
* **ABR**: Average BitRate，平均比特率 是VBR的一种插值参数。ABR在指定的文件大小内，以每50帧 （30帧约1秒）为一段，低频和不敏感频率使用相对低的流量，高频和大动态表现时使用高流量，可以做为VBR和CBR的一种折衷选择。  
* **CBR**: Constant BitRate，是以恒定比特率方式进行编码，有Motion发生时，由于码率恒定，只能通过增大QP来减少码字大小，图像质量变差，当场景静止时，图像质量又变好，因此图像质量不稳定。优点是压缩速度快，缺点是每秒流量都相同容易导致空间浪费。  
* **CVBR**: Constrained Variable it Rate，VBR的一种改进，兼顾了CBR和VBR的优点：在图像内容静止时，节省带宽，有Motion发生时，利用前期节省的带宽来尽可能的提高图像质量，达到同时兼顾带宽和图像质量的目的。这种方法通常会让用户输入最大码率和最小码率，静止时，码率稳定在最小码率，运动时，码率大于最小码率，但是又不超过最大码率。

## Profile level
1. BP-Baseline Profile：基本画质。支持I/P 帧，只支持无交错（Progressive）和CAVLC；
2. EP-Extended profile：进阶画质。支持I/P/B/SP/SI 帧，只支持无交错（Progressive）和CAVLC；
3. MP-Main profile：主流画质。提供I/P/B 帧，支持无交错（Progressive）和交错（Interlaced），也支持CAVLC 和CABAC 的支持；
4. HP-High profile：高级画质。在main Profile 的基础上增加了8x8内部预测、自定义量化、无损视频编码和更多的YUV 格式。

# 2 代码实现
FFmpeg的avcodec用于视频编码解码工作，里面包含了几个重要的数据结构：
* AVCodec: 编码器或解码器。
* AVCodecContext: 编解码上下文。记录了当前编解码操作使用的编解码器ID、比特率、分辨率、FPS、帧采样格式、压缩等级等配置信息。在编解码过程中，需要通过这个对象作为参数进行编码或解码操作。
* AVFrame: 帧信息。存储了一帧数据的元数据信息（分辨率、帧取样格式等），以及图像数据。
* AVPacket: 压缩编码数据包，后面简称“压缩包”。编码过程中，会向AVCodecContext对象传递每一帧数据，但其内在是先把这些帧数据缓存起来，直到填满缓冲区时才一次过对缓冲区中的数据压缩成压缩包。解码过程是编码过程的逆操作，对缓冲区中的压缩包解码后得到帧数据。
* AVCodecParserContext: 编码解析器上下文。这个对象在解码的时候用到，用于把编码压缩数据的数据流切分成若干帧的编码压缩数据，凑到足够多的切分数据后就可以组成压缩包，再对压缩包进行解码获取到帧数据。

其中，AVCodecContext、AVFrame、AVPacket 在不需要使用时，要调用相应的free函数释放掉内存。

## 2.1 视频编码
编码过程的步骤可分为：
1. 通过名称获取avcodec提供的编码器(AVCodec)。
2. 通过AVCodec创建编码上下文(AVCodecContext)。
3. 配置AVCodecContext（比特率、分辨率、FPS、GOP、帧取样格式等）。
4. 初始化AVPacket。
5. 初始化AVFrame，配置相应参数：帧取样格式、分辨率。
6. 根据第5步的配置信息，申请相应大小的帧数据存储空间。
7. 绘制帧内容。
8. 把绘制好的单帧数据发送给AVCodecContext。
9. 检查编码器缓冲区是否已经填满。如果是，则将缓冲区压缩后的内容写入文件。
10. 回到第7步绘制下一帧数据。
11. 若没有下一帧数据可绘制，则向AVCodecContext发送一个NULL帧，将会把剩余还未填满缓冲区的数据进行压缩，再把这些剩余的数据也写到文件中。
12. 释放AVCodecContext，AVPacket，AVFrame资源。

下面将给出对应上面步骤的代码一一进行讲解，本例子使用程序绘制25个图片帧编码成视频输出到指定的文件中，输出的视频时长为1秒（25fps）。

```cpp
/* C++编译时要添加 extern "C" */
extern "C" {
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>
}

int main(int argc, char **argv) {
    const char *filename, *codec_name;
    const AVCodec *codec;
    AVCodecContext *c;

    uint8_t endcode[] = {0, 0, 1, 0xb7};

    // 从进程运行参数中获取编码文件名与编码器名称
    filename = argv[1];
    codec_name = argv[2];

    // 根据名称查询编码器
    codec = avcodec_find_encoder_by_name(codec_name);
    if (!codec) {
        fprintf(stderr, "Codec '%s' not found\n", codec_name);
        exit(1);
    }

    // 通过编码器创建编码上下文
    c = avcodec_alloc_context3(codec);
    if (!c) {
        fprintf(stderr, "Could not allocate video codec context\n");
        exit(1);
    }

    ...
}
```

FFmpeg中的函数实现都在C语言编写的动态链接库中，如果用C++编写代码则需要把引入FFmpeg头文件的语句包括在 extern "C"{ ... } 的大括号中。**avcodec_find_encoder_by_name()** 函数用于根据名称获取对应的编码器，可以在终端上执行 "ffmpeg -codecs" 指令查看电脑上提供的编解码器，把名称作为参数传进 **avcodec_find_encoder_by_name()** 函数即可在代码中调用。当获取编码器失败时，**avcodec_find_encoder_by_name()** 函数将返回 **NULL**。而若我们获取编码器成功后，就可以把编码器作为参数传给 **avcodec_alloc_context3()** 函数来创建编码上下文对象(AVCodecContext)，同样若 **avcodec_alloc_context3()** 函数创建编码上下文失败，则会返回 **NULL**。

然后需要对创建的编码上下文进行配置，代码如下：

```cpp
// 设置比特率
c->bit_rate = 400000;
// 设置分辨率
c->width = 352;
c->height = 288;
// 设置fps，time_base 是 framerate 的倒数
c->time_base = (AVRational) {1, 25};
c->framerate = (AVRational) {25, 1};
// GOP大小
c->gop_size = 10;
// 两个非B帧之间的B帧最大数目（设为0表示不会有B帧）
c->max_b_frames = 0;
// 帧采样格式
c->pix_fmt = AV_PIX_FMT_YUV420P;

// H264编码时还可以调节编码速度从而调整压缩质量，这里把编码速度设置为slow
if (codec->id == AV_CODEC_ID_H264)
    av_opt_set(c->priv_data, "preset", "slow", 0);
```

上述大部分配置参数的概念已经在 **基础概念** 章节做过介绍，这里不再详述。 唯一要说明的是 **av_opt_set()** 函数，它是在 **libavutil/opt.h** 头文件中引入，它的作用是可以配置AVCodecContext的 **priv_data** 成员变量属性。实际上，AVCodecContext对象是任何编码器的编码上下文抽象数据结构，这个数据结构中已经定义了大部分通用参数，但各自特定的配置参数保存在AVCodecContext的 **priv_data** 成员变量中，它是一个 void 指针，因为不同编码器对应编码上下文中 **priv_data** 成员变量的数据结构会有所不同，需要通过 **av_opt_set()** 函数来设置这些特定参数。H264 编码器对应的 **priv_data** 是 **X264Context** 数据结构（[详看源码](https://github.com/FFmpeg/FFmpeg/blob/n4.1.1/libavcodec/libx264.c)）， "av_opt_set(c->priv_data, "preset", "slow", 0)" 语句把 **X264Context** 数据结构中的 **preset** 成员变量的值设为字符串 "slow"。 

```cpp
FILE *f;
int ret;

...

// 打开编码上下文
ret = avcodec_open2(c, codec, NULL);
if (ret < 0) {
    fprintf(stderr, "Could not open codec: %s\n", av_err2str(ret));
    exit(1);
}

// 打开文件流
f = fopen(filename, "wb");
if (!f) {
    fprintf(stderr, "Could not open %s\n", filename);
    exit(1);
}
```

因为整个过程都是流式数据处理，需要像打开文件流那样把编码上下文也变为打开状态（采用 **avcodec_open2()** 函数）。在编码过程中，会先对缓冲区数据进行压缩，然后把压缩后的数据写入的文件中，所以我们也要打开文件流供编码数据写入。

```cpp
AVFrame *frame;
AVPacket *pkt;

...

// 创建维护缓冲区数据的AVPacket对象
pkt = av_packet_alloc();
if (!pkt)
    exit(1);

// 创建维护帧元数据的AVFrame对象
frame = av_frame_alloc();
if (!frame) {
    fprintf(stderr, "Could not allocate video frame\n");
    exit(1);
}
// 设置帧取样格式
frame->format = c->pix_fmt;
// 设置帧的分辨率
frame->width = c->width;
frame->height = c->height;

// 根据AVFrame的设置分配对应大小的缓存空间
ret = av_frame_get_buffer(frame, 0);
if (ret < 0) {
    fprintf(stderr, "Could not allocate the video frame data\n");
    exit(1);
}
```

**av_packet_alloc()** 函数用于创建 AVPacket 对象，**av_frame_alloc()** 函数用于创建 AVFrame 对象，此时 AVFrame 还不知道需要多大的帧存储空间，需要对 AVFrame 对象指定申请帧数据存储空间需要用到的配置参数（如：帧取样格式 format，分辨率 width * height），然后再调用 **av_frame_get_buffer()** 函数申请帧数据存储空间，

完成上述操作后，就可以开始绘制每一帧的像素值，然后进行编码。

```cpp
/* 采用 4:2:0 取样格式，绘制并编码一秒钟的视频帧数据(25fps) */
for (i = 0; i < 25; i++) {
    // 绘制 Y 空间数据
    for (y = 0; y < c->height; y++) {
        for (x = 0; x < c->width; x++) {
            frame->data[0][y * frame->linesize[0] + x] = x + y + i * 3;
        }
    }

    // 绘制 Cb 与 Cr 空间数据
    for (y = 0; y < c->height / 2; y++) {
        for (x = 0; x < c->width / 2; x++) {
            frame->data[1][y * frame->linesize[1] + x] = 128 + y + i * 2;
            frame->data[2][y * frame->linesize[2] + x] = 64 + x + i * 5;
        }
    }

    // 帧位置，该帧的播放时间为：pts * time_base
    frame->pts = i;

    // 将本帧图片发送到编码器缓冲区，必要时压缩缓冲区中的所有帧数据，并写入到文件中
    encode(c, frame, pkt, f);
}

// 最后一帧设为NULL，表示到达流末端，这个操作会把剩余未压缩的缓冲区数据编码到文件中。
encode(c, NULL, pkt, f);
```

编码过程中，会向AVCodecContext对象传递每一帧数据，但其内在是先把这些帧数据缓存起来，直到填满缓冲区时才一次过对缓冲区中的数据进行压缩。如果已经把所有帧数据发送给了编码器，编码器缓冲区中可能因尚未填满缓冲区而剩余了一些未压缩的帧数据，需要向编码上下文发送一个 **NULL** 帧通知编码器把剩余的缓冲区数据进行压缩，并写入到文件中。

**encode()** 是我们自定义的函数，它的作用是把帧数据发送给编码上下文，当编码器缓冲区对数据进行压缩后，把压缩后的数据写入到文件中。下面给出 **encode()** 函数的过程代码：

```cpp
static void encode(AVCodecContext *enc_ctx, AVFrame *frame, AVPacket *pkt, FILE *outfile) {
    int ret;

    // 发送帧数据到编码器
    ret = avcodec_send_frame(enc_ctx, frame);
    if (ret < 0) {
        fprintf(stderr, "Error sending a frame for encoding\n");
        exit(1);
    }

    while (ret >= 0) {
        // 尝试获取编码器缓冲区压缩后的数据
        ret = avcodec_receive_packet(enc_ctx, pkt);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
            return;
        else if (ret < 0) {
            fprintf(stderr, "Error during encoding\n");
            exit(1);
        }

        // 把压缩后的数据写入文件中
        fwrite(pkt->data, 1, pkt->size, outfile);
        // 释放AVPacket中的数据
        av_packet_unref(pkt);
    }
}
```

**avcodec_send_frame()** 函数把帧数据传入编码器，通过 **avcodec_receive_packet()** 函数获取编码器缓冲区的数据，根据缓冲区状态不同，这个函数会有不同的返回值（下面有详细介绍）。若 **avcodec_receive_packet()** 函数返回值是 0，说明缓冲区已被填满并进行了数据压缩操作，压缩后的数据已经返回到 **avcodec_receive_packet()** 函数第三个参数的 AVPacket 对象中，然后通过 **fwrite()** 函数把压缩后的数据写入到文件中，之后还要调用 **av_packet_unref()** 函数释放 AVPacket 中的数据内存。

**avcodec_receive_packet()** 函数返回值说明：
* 当函数返回 0，表示编码成功，此时pkt中存储了压缩帧数据，过后通过fwrite把数据写入文件；
* 当函数返回 AVERROR(EAGAIN)，表示编码器需要接收更多的输入帧以填满缓冲区为止，再对缓冲区进行压缩；
* 当函数返回 AVERROR_EOF，表示到达了数据流末尾，没有可编码的数据了；
* 当函数返回其它结果，代表编码或解码过程中发生了异常。

到这里，已经把所有的帧数据编码到文件中，但按照MPEG标准，还需要在文件末尾添加4个字节的**结束序列码**，结束序列码在下面代码中给出：

```cpp
int main(int argc, char **argv) {
    // 结束序列码
    uint8_t endcode[] = {0, 0, 1, 0xb7};

    ...

    // 按照MPEG标准，需要在文件末尾添加结束序列码
    fwrite(endcode, 1, sizeof(endcode), f);
    fclose(f);

    // 释放AVCodecContext
    avcodec_free_context(&c);
    // 释放AVFrame
    av_frame_free(&frame);
    // 释放AVPacket
    av_packet_free(&pkt);

    return 0;
}
```

结束序列码以4个字节组成，用C语言的 **uint8_t** 类型表示为 **{0, 0, 1, 0xb7}**。把它写入到文件末尾，然后关闭文件流，就算是完成了视频编码到文件的工作流程，最后记得调用 **avcodec_free_context()、av_frame_free()、av_packet_free()** 分别释放 AVCodecContext，AVPacket，AVFrame 的资源。

## 2.2 视频解码
解码过程的步骤可分为：
1. 通过名称获取avcodec提供的解码器(AVCodec)。
2. 通过AVCodec创建编码解析器上下文(AVCodecParserContext)。
3. 通过AVCodec创建解码上下文(AVCodecContext)。
4. 初始化AVPacket与AVFrame，解码时不需要给AVFrame配置，也不需要申请帧数据内存空间。
5. 读取视频文件流，对从文件流读取的裸编码数进行压缩帧切分，并把切分后的压缩帧放入解码器缓冲区。
6. 检查解码器缓冲区中是否有足够的数据形成压缩包，如果有，则把压缩包解码，并把解码帧数据写入到文件中。
7. 直到视频文件流全部读取完毕，最后需要清空解码器缓冲区，把剩下的压缩帧数据也解码出来。
8. 释放AVCodecContext，AVCodecParserContext，AVPacket，AVFrame资源。

上面说的压缩帧与解码帧不同在于：压缩帧虽然按帧划分了数据结构，但是保存的帧数据依然是压缩过的，还不能直接提取使用；解码帧指的是对压缩帧解压后的帧数据，可以直接获取帧像素数据。

编写示例，对 **视频编码** 章节生成的视频进行解码，获取视频解码后的每帧数据，并保存到文件中。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* C++编译时要添加 extern "C" */
extern "C" {
#include <libavcodec/avcodec.h>
}

// 文件流缓存大小
#define INBUF_SIZE 4096

int main(int argc, char **argv) {
    // 进程运行参数中指定视频路径，输出图片帧文件路径，解码器名称
    const char* filename = argv[1];
    const char* outfilename = argv[2];
    const char* codec_name = argv[3];
    
    /* 根据名称查询解码器 */
    AVCodec *codec = avcodec_find_decoder_by_name(codec_name);
    if (!codec) {
        fprintf(stderr, "Codec '%s' not found\n", codec_name);
        exit(1);
    }

    // 初始化AVCodecParserContext
    AVCodecParserContext *parser = av_parser_init(codec->id);
    if (!parser) {
        fprintf(stderr, "parser not found\n");
        exit(1);
    }

    /*
     * 根据解码器创建解码上下文
     * 某些解码器（如：msmpeg4和mpeg4）必须初始化图像分辨率信息，
     * 因为它们的数据流中没有保存分辨率信息
     */
    AVCodecContext *c = avcodec_alloc_context3(codec);
    if (!c) {
        fprintf(stderr, "Could not allocate video codec context\n");
        exit(1);
    }

    ...
}
```

按照步骤先初始化 AVCodec、AVCodecParserContext、AVCodecContext 对象。需要注意的是，根据名称获取解码器的函数是 **avcodec_find_decoder_by_name()**。创建AVCodecParserContext对象的函数是**av_parser_init()**，它需要传入解码器ID作为参数。一般情况下，解码时是不需要为AVCodecContext配置信息。某些解码器（如：msmpeg4和mpeg4）必须初始化图像分辨率信息，因为它们的数据流中没有保存分辨率信息。

然后初始化AVPacket与AVFrame。

```cpp
// 初始化AVFrame
AVFrame *frame = av_frame_alloc();
if (!frame) {
    fprintf(stderr, "Could not allocate video frame\n");
    exit(1);
}

// 初始化AVPacket
AVPacket *pkt = av_packet_alloc();
if (!pkt)
    exit(1);
```

与编码过程不同的是，解码时，不需要为AVFrame配置分辨率，也不需要申请帧数据内存空间。

接下来以二进制只读方式开启视频文件流，读取视频文件内容。与编码过程一样，也需要通过 **avcodec_open2()** 函数打开解码上下文。

```cpp
// 打开待解码视频的文件流
FILE *f = fopen(filename, "rb");
if (!f) {
    fprintf(stderr, "Could not open %s\n", filename);
    exit(1);
}

// 打开解码上下文
if (avcodec_open2(c, codec, NULL) < 0) {
    fprintf(stderr, "Could not open codec\n");
    exit(1);
}
```

接下来的代码演示解码步骤5,6,7：
```cpp
// 输入缓存，除了指定缓冲区大小，还要多加一个PADDING_SIZE的大小（av_parser_parse2()函数的输入数组需要腾出这个空间）
uint8_t inbuf[INBUF_SIZE + AV_INPUT_BUFFER_PADDING_SIZE];
// 对 PADDING 部分的缓存值都置为0，避免数据干扰
memset(inbuf + INBUF_SIZE, 0, AV_INPUT_BUFFER_PADDING_SIZE);
// 作为inbuf数组上的游标指针使用
uint8_t *data;
// 用于记录fread()函数返回值
size_t data_size;
// 用于记录av_parser_parse2()函数返回值
int ret;

// 开始对视频进行解码
while (!feof(f)) {
    // 从视频文件流中读取裸数据到缓存中
    data_size = fread(inbuf, 1, INBUF_SIZE, f);
    if (!data_size)
        break;

    data = inbuf;
    while (data_size > 0) {
        /*
         * 使用parser将缓存中的裸数据切分成若干帧的压缩编码数据。
         * pkt->size为0时，说明当前解码器缓冲区中的压缩编码数据不足以形成一个压缩包进行解码，还需要读更多的压缩编码数据。
         * 与编码过程相反，编码时需要足够多的帧填满缓冲区再压缩成压缩包，这里需要足够多的压缩编码数据形成一个压缩包。
         */
        ret = av_parser_parse2(parser, c, &pkt->data, &pkt->size,
                                data, data_size, AV_NOPTS_VALUE, AV_NOPTS_VALUE, 0);
        if (ret < 0) {
            fprintf(stderr, "Error while parsing\n");
            exit(1);
        }
        data += ret;
        data_size -= ret;

        /*
         * 若足够形成压缩包，则对压缩包中的压缩编码数据进行解码。
         * 把解码后的每帧数据输出到 "outfilename-{idx}" 文件中，idx 指帧位置下标。
         */
        if (pkt->size)
            decode(c, frame, pkt, outfilename);
    }
}

// 向解码器发送一个 NULL 压缩包，表示告知解码器要清空缓冲区，把还未解码的帧数据一并解码返回，然后发送EOS信号。
decode(c, frame, NULL, outfilename);

// 关闭视频文件流
fclose(f);
```

**av_parser_parse2()** 的参数定义如下：
* 第一个参数：AVCodecParserContext对象。
* 第二个参数：AVCodecContext对象。
* 第三个参数：若压缩帧切分完毕，参数将会被修改为压缩包数据。否则，修改为NULL。
* 第四个参数：若压缩帧切分完毕，参数将会被修改为压缩包的数据长度。否则，修改为0。
* 第五个参数：输入视频流的编码压缩数据。（本例子是从文件流中读取）
* 第六个参数：输入视频流裸数据的长度。
* 第七个参数：指定输入的编码压缩数据代表的帧下标区间段的起始位置。若不能给出，则传 libavutil/avutil.h 文件下的 **AV_NOPTS_VALUE** 常量，一般传**AV_NOPTS_VALUE**即可。
* 第八个参数：指定输入的编码压缩数据代表的帧下标区间段的结束位置。若不能给出，则传 libavutil/avutil.h 文件下的 **AV_NOPTS_VALUE** 常量，一般传**AV_NOPTS_VALUE**即可。
* 第九个参数：指定从输入视频流的编码压缩数据中的第几个下标开始读取。
* 返回值：表示本次输入的数据中有多少个字节被使用上。

从第四个参数定义可以得知，当该参数值被修改为非0值，则表示帧切分完毕，可以提取压缩包内容进行解码。