[![license](https://img.shields.io/badge/license-BSD--3--Clause-blue.svg)](https://img.shields.io/badge/license-BSD--3--Clause-blue.svg)

Будучи в поисках учебного пособия/книги, которые научили бы меня использовать [FFmpeg](https://www.ffmpeg.org/) в качестве библиотеки (aka libav) я нашел статью [«How to Write a Video Player in Less Than 1000 Lines»](http://dranger.com/ffmpeg/).
К сожалению, информация там устарела, поэтому я решил написать свой материал

Большая часть представленого здесь кода будет на C, **но не волнуйтесь**: вы можете легко понять и применить ее на предпочитаемом вами языке.
FFmpeg libav имеет множество биндингов для множества языков, таких как [python](https://mikeboers.github.io/PyAV/), [go](https://github.com/imkira/go-libav) и даже если ваш любимый язык не имеет готовый биндинг, вы все равно можете использовать его через `ffi` (вот пример с [Lua](https://github.com/daurnimator/ffmpeg-lua-ffi/blob/master/init.lua)).

Мы начнем с введения о том, что такое видео, аудио, кодек и контейнер, а затем мы перейдем к ускоренному курсу о том, как использовать приложение для командной строки `FFmpeg`, и, в конце, мы напишем код. Не стесняйтесь переходить напрямую в [](http://newmediarockstars.com/wp-content/uploads/2015/11/nintendo-direct-iwata.jpg) раздел [Learn FFmpeg libav the Hard Way.](#learn-ffmpeg-libav-the-hard-way)

Некоторые люди считают, что потоковое видео в Интернете - это будущее традиционного телевидения. В любом случае, FFmpeg - стоит изучения

__Оглавление__

* [Вступление](#вступление)
  * [video - what you see!](#video---what-you-see)
  * [audio - what you listen!](#audio---what-you-listen)
  * [codec - shrinking data](#codec---shrinking-data)
  * [container - a comfy place for audio and video](#container---a-comfy-place-for-audio-and-video)
* [FFmpeg - command line](#ffmpeg---command-line)
  * [FFmpeg command line tool 101](#ffmpeg-command-line-tool-101)
* [Common video operations](#common-video-operations)
  * [Transcoding](#transcoding)
  * [Transmuxing](#transmuxing)
  * [Transrating](#transrating)
  * [Transsizing](#transsizing)
  * [Bonus Round: Adaptive Streaming](#bonus-round-adaptive-streaming)
  * [Going beyond](#going-beyond)
* [Learn FFmpeg libav the Hard Way](#learn-ffmpeg-libav-the-hard-way)
  * [Chapter 0 - The infamous hello world](#chapter-0---the-infamous-hello-world)
    * [FFmpeg libav architecture](#ffmpeg-libav-architecture)
    * [Chapter 0 - code walkthrough](#chapter-0---code-walkthrough)

# Вступление

## видео - то, что видите!

Если серию последовательных карнинок поочередно менять с определенной переодичностью (предположим, [24 кадра в секунду](ttps://www.filmindependent.org/blog/hacking-film-24-frames-per-second/)), создастся [иллюзия движения](https://ru.wikipedia.org/wiki/Инерция_зрения).
В итоге, это простая идея легла в основу видео: **серия картинок/кадров меняющихся с определенной частотой**.

<img src="https://upload.wikimedia.org/wikipedia/commons/1/1f/Linnet_kineograph_1886.jpg" height="280"></img>

Иллюстрация Zeitgenössische (1886)

## аудио - то, что вы слышите!

Несмотря на то, что видеодорожка сама по себе может вызывать большое количество чувств, добавление к ней звука приносит заставляет приносить ее больше удовольствия.

Звук - это вибрация, распростроняющаяся в виде упругих волн механических колебаний твёрдой, жидкой или газообразной среде (например, воздухе).

> В цифровой аудиосистеме микрофон преобразует звук в аналоговый электрический сигнал, затем аналого-цифровой преобразователь (АЦП) - с использованием [импульсно-кодовой модуляции (ИКМ)](https://ru.wikipedia.org/wiki/Импульсно-кодовая_модуляция) преобразует аналоговый сигнал в цифровой сигнал.

![преобразование аналогово аудио сингала в цифровое](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c7/CPT-Sound-ADC-DAC.svg/640px-CPT-Sound-ADC-DAC.svg.png "преобразование аналогово аудио сингала в цифровое")
>[Источник](https://commons.wikimedia.org/wiki/File:CPT-Sound-ADC-DAC.svg)

## кодек - сжимаем данные

> **Видеокодек** — программа/алгоритм сжатия (то есть уменьшения размера) видеоданных (видеофайла, видеопотока) и восстановления сжатых данных. Кодек — файл-формула, которая определяет, каким образом можно «упаковать» видеоконтент и, соответственно, воспроизвести видео.
> https://ru.wikipedia.org/wiki/Видеокодек

Но если мы решим упаковать миллионы изображений в один файл и назвать это фильмом, мы получим огромный файл. Давайте сделаем некоторые подсчеты:

Допустим, мы создаем видео с разрешением `1080 x 1920` (высота x ширина), тратя `3 байта` на каждый пиксель, чтобы закодировать цвет (так называемый [TrueColor](https://ru.wikipedia.org/wiki/TrueColor), который дает нам 16,777,216 разных цветов) и это видео продолжительностью `30 минут` содержит `24 кадра в секунду`.

```c
toppf = 1080 * 1920 //total_of_pixels_per_frame
cpp = 3 //cost_per_pixel
tis = 30 * 60 //time_in_seconds
fps = 24 //frames_per_second

required_storage = tis * fps * toppf * cpp
```

Для видео понадобилость бы примерно `250.28GB` места или `1.11Gbps` пропускной способности! Поэтому, если необходимость в использовании [кодека](https://github.com/leandromoreira/digital_video_introduction#how-does-a-video-codec-work).


## контейнер - уютное место для вашего видео и аудио

> A container or wrapper format is a metafile format whose specification describes how different elements of data and metadata coexist in a computer file.
> https://en.wikipedia.org/wiki/Digital_container_format

**Отдельный файл, содержащий все потоки** (в основном аудио и видео), а также предоставляющий **синхронизацию и основные метаданные**, такие как название, разрешение и т.д.

Обычно мы можем определить формат файла, посмотрев на его расширение: например, «video.webm» - это, вероятно, видео использующее контейнер [`webm`](https://www.webmproject.org/).

![container](/img/container.png)

# Консольная утилита FFmpeg

> Готовое, кросс-платформенное решение для записи, конвертирования и передачи аудио и видео.

Для работы с мультимедиа мы можем использовать чудесную утилиту/библиотеку под названием [FFmpeg](https://www.ffmpeg.org/). Скорее всего, вы уже знаете о ней, или даже используете ее прямо или косвенно (вы используете [Chrome?](https://www.chromium.org/developers/design-documents/video)).

В него входит утилита для командной строки, именнуемая `ffmpeg`, очень простая но мощная программа.
Например, вы можете конвертировать из `mp4` в контейнер` avi`, просто используя следующую команду:

```bash
$ ffmpeg -i input.mp4 output.avi
```

Мы просто совершили **remuxing**, который конвертирует из одного контейнера в другой.
Технически, FFmpeg также может делать транскодинг, но о нем мы поговорим немного позже.

## Курс молодого бойца по консольной утилите FFmpeg

FFmpeg поставляется с хорошей [документацией (англ.)](https://www.ffmpeg.org/ffmpeg.html) которая прекрасно объясняет, как он работает.

В двух словах, утилита FFmpeg для работы ожидает следующий формат аргументов `ffmpeg {1} {2} -i {3} {4} {5}`, где:

1. глобальные параметры
2. параметры входного файла
3. адрес входного файла
4. параметры выходного файла
5. адрес выходного файла

Части 2, 3, 4 и 5 могут использованы столько раз, сколько это необходимо.
Этот формат лучше понимается на примерах:

``` bash
# Предупреждение: размер файла около 300MB
$ wget -O bunny_1080p_60fps.mp4 http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_60fps_normal.mp4

$ ffmpeg \
-y \ # {1} глобальные параметры
-c:a libfdk_aac -c:v libx264 \ # {2} параметры входного файла
-i bunny_1080p_60fps.mp4 \ # {3} адрес входного файла
-c:v libvpx-vp9 -c:a libvorbis \ # {4} параметры выходного файла
bunny_1080p_60fps_vp9.webm # {5} адрес выходного файла
```

This command takes an input file `mp4` containing two streams (an audio encoded with `aac` CODEC and a video encoded using `h264` CODEC) and convert it to `webm`, changing its audio and video CODECs too.

We could simplify the command above but then be aware that FFmpeg will adopt or guess the default values for you.
For instance when you just type `ffmpeg -i input.avi output.mp4` what audio/video CODEC does it use to produce the `output.mp4`?

Werner Robitza wrote a must read/execute [tutorial about encoding and editing with FFmpeg](http://slhck.info/ffmpeg-encoding-course/#/).

# Common video operations

While working with audio/video we usually do a set of tasks with the media.

## Transcoding

![transcoding](/img/transcoding.png)

**What?** the act of converting one of the streams (audio or video) from one CODEC to another one.

**Why?** sometimes some devices (TVs, smartphones, console and etc) doesn't support X but Y and newer CODECs provide better compression rate.

**How?** converting an `H264` (AVC) video to an `H265` (HEVC).
```bash
$ ffmpeg \
-i bunny_1080p_60fps.mp4 \
-c:v libx265 \
bunny_1080p_60fps_h265.mp4
```

## Transmuxing

![transmuxing](/img/transmuxing.png)

**What?** the act of converting from one format (container) to another one.

**Why?** sometimes some devices (TVs, smartphones, console and etc) doesn't support X but Y and sometimes newer containers provide modern required features.

**How?** converting a `mp4` to a `webm`.
```bash
$ ffmpeg \
-i bunny_1080p_60fps.mp4 \
-c copy \ # just saying to ffmpeg to skip encoding
bunny_1080p_60fps.webm
```

## Transrating

![transrating](/img/transrating.png)

**What?** the act of changing the bit rate, or producing other renditions.

**Why?** people will try to watch your video in a `2G` (edge) connection using a less powerful smartphone or in a `fiber` Internet connection on their 4K TVs therefore you should offer more than on rendition of the same video with different bit rate.

**How?** producing a rendition with bit rate between 3856K and 2000K.
```bash
$ ffmpeg \
-i bunny_1080p_60fps.mp4 \
-minrate 964K -maxrate 3856K -bufsize 2000K \
bunny_1080p_60fps_transrating_964_3856.mp4
```

Usually we'll be using transrating with transsizing. Werner Robitza wrote another must read/execute [series of posts about FFmpeg rate control](http://slhck.info/posts/).

## Transsizing

![transsizing](/img/transsizing.png)

**What?** the act of converting from one resolution to another one. As said before transsizing is often used with transrating.

**Why?** reasons are about the same as for the transrating.

**How?** converting a `1080p` to a `480p` resolution.
```bash
$ ffmpeg \
-i bunny_1080p_60fps.mp4 \
-vf scale=480:-1 \
bunny_1080p_60fps_transsizing_480.mp4
```

## Bonus Round: Adaptive Streaming

![adaptive streaming](/img/adaptive-streaming.png)

**What?** the act of producing many resolutions (bit rates) and split the media into chunks and serve them via http.

**Why?** to provide a flexible media that can be watched on a low end smartphone or on a 4K TV, it's also easy to scale and deploy but it can add latency.

**How?** creating an adaptive WebM using DASH.
```bash
# video streams
$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 160x90 -b:v 250k -keyint_min 150 -g 150 -an -f webm -dash 1 video_160x90_250k.webm

$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 320x180 -b:v 500k -keyint_min 150 -g 150 -an -f webm -dash 1 video_320x180_500k.webm

$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 640x360 -b:v 750k -keyint_min 150 -g 150 -an -f webm -dash 1 video_640x360_750k.webm

$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 640x360 -b:v 1000k -keyint_min 150 -g 150 -an -f webm -dash 1 video_640x360_1000k.webm

$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 1280x720 -b:v 1500k -keyint_min 150 -g 150 -an -f webm -dash 1 video_1280x720_1500k.webm

# audio streams
$ ffmpeg -i bunny_1080p_60fps.mp4 -c:a libvorbis -b:a 128k -vn -f webm -dash 1 audio_128k.webm

# the DASH manifest
$ ffmpeg \
 -f webm_dash_manifest -i video_160x90_250k.webm \
 -f webm_dash_manifest -i video_320x180_500k.webm \
 -f webm_dash_manifest -i video_640x360_750k.webm \
 -f webm_dash_manifest -i video_640x360_1000k.webm \
 -f webm_dash_manifest -i video_1280x720_500k.webm \
 -f webm_dash_manifest -i audio_128k.webm \
 -c copy -map 0 -map 1 -map 2 -map 3 -map 4 -map 5 \
 -f webm_dash_manifest \
 -adaptation_sets "id=0,streams=0,1,2,3,4 id=1,streams=5" \
 manifest.mpd
```

PS: I stole this example from the [Instructions to playback Adaptive WebM using DASH](http://wiki.webmproject.org/adaptive-streaming/instructions-to-playback-adaptive-webm-using-dash)

## Going beyond

There are [many and many other usages for FFmpeg](https://github.com/leandromoreira/digital_video_introduction/blob/master/encoding_pratical_examples.md#split-and-merge-smoothly).
I use it in conjunction with *iMovie* to produce/edit some videos for YouTube and you can certainly use it professionally.

# Learn FFmpeg libav the Hard Way

> Don't you wonder sometimes 'bout sound and vision?
> **David Robert Jones**

Since the [FFmpeg](#ffmpeg---command-line) is so useful as a command line tool to do essential tasks over the media files, how can we use it in our programs?

FFmpeg is [composed by several libraries](https://www.ffmpeg.org/doxygen/trunk/index.html) that can be integrated into our own programs.
Usually, when you install FFmpeg, it installs automatically all these libraries. I'll be referring to the set of these libraries as **FFmpeg libav**.

> This title is a homage to Zed Shaw's series [Learn X the Hard Way](https://learncodethehardway.org/), particularly his book Learn C the Hard Way.

## Chapter 0 - The infamous hello world
This hello world actually won't show the message `"hello world"` in the terminal :tongue:
Instead we're going to **print out information about the video**, things like its format (container), duration, resolution, audio channels and, in the end, we'll **decode some frames and save them as image files**.

### FFmpeg libav architecture

But before we start to code, let's learn how **FFmpeg libav architecture** works and how its components communicate with others.

Here's a diagram of the process of decoding a video:

![ffmpeg libav architecture - decoding process](/img/decoding.png)

You'll first need to load your media file into a component called [`AVFormatContext`](https://ffmpeg.org/doxygen/trunk/structAVFormatContext.html) (the video container is also known as format).
It actually doesn't fully load the whole file: it often only reads the header.

Once we loaded the minimal **header of our container**, we can access its streams (think of them as a rudimentary audio and video data).
Each stream will be available in a component called [`AVStream`](https://ffmpeg.org/doxygen/trunk/structAVStream.html).

> Stream is a fancy name for a continuous flow of data.

Suppose our video has two streams: an audio encoded with [AAC CODEC](https://en.wikipedia.org/wiki/Advanced_Audio_Coding) and a video encoded with [H264 (AVC) CODEC](https://en.wikipedia.org/wiki/H.264/MPEG-4_AVC). From each stream we can extract **pieces (slices) of data** called packets that will be loaded into components named [`AVPacket`](https://ffmpeg.org/doxygen/trunk/structAVPacket.html).

The **data inside the packets are still coded** (compressed) and in order to decode the packets, we need to pass them to a specific [`AVCodec`](https://ffmpeg.org/doxygen/trunk/structAVCodec.html).

The `AVCodec` will decode them into [`AVFrame`](https://ffmpeg.org/doxygen/trunk/structAVFrame.html) and finally, this component gives us **the uncompressed frame**.  Noticed that the same terminology/process is used either by audio and video stream.

### Chapter 0 - code walkthrough

> #### TLDR; show me the [code](/0_hello_world.c) and execution.
> ```bash
> # WARNING: this file is around 300MB
> $ make
> ```

We'll skip some details, but don't worry: the [source code is available at github](/0_hello_world.c).

The first thing we need to do is to register all the codecs, formats and protocols.
To do it, we just need to call the function [`av_register_all`](http://ffmpeg.org/doxygen/trunk/group__lavf__core.html#ga917265caec45ef5a0646356ed1a507e3):

```c
av_register_all();
```

Now we're going to allocate memory to the component [`AVFormatContext`](http://ffmpeg.org/doxygen/trunk/structAVFormatContext.html) that will hold  information about the format (container).

```c
AVFormatContext *pFormatContext = avformat_alloc_context();
```

Now we're going to open the file and read its header and fill the `AVFormatContext` with minimal information about the format (notice that usually the codecs are not opened).
The function used to do this is [`avformat_open_input`](http://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#ga31d601155e9035d5b0e7efedc894ee49). It expects an `AVFormatContext`, a `filename` and two optional arguments: the [`AVInputFormat`](https://ffmpeg.org/doxygen/trunk/structAVInputFormat.html) (if you pass `NULL`, FFmpeg will guess the format) and the [`AVDictionary`](https://ffmpeg.org/doxygen/trunk/structAVDictionary.html) (which are the options to the demuxer).

```c
avformat_open_input(&pFormatContext, filename, NULL, NULL);
```

We can print the format name and the media duration:

```c
printf("Format %s, duration %lld us", pFormatContext->iformat->long_name, pFormatContext->duration);
```

To access the `streams`, we need to read data from the media. The function [`avformat_find_stream_info`](https://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#gad42172e27cddafb81096939783b157bb) does that.
Now, the `pFormatContext->nb_streams` will hold the amount of streams and the `pFormatContext->streams[i]` will give us the `i` stream (an [`AVStream`](https://ffmpeg.org/doxygen/trunk/structAVStream.html)).

```c
avformat_find_stream_info(pFormatContext,  NULL);
```

Now we'll loop through all the streams.

```c
for (int i = 0; i < pFormatContext->nb_streams; i++)
{
  //
}
```

For each stream, we're going to keep the [`AVCodecParameters`](https://ffmpeg.org/doxygen/trunk/structAVCodecParameters.html), which describes the properties of a codec used by the stream `i`.

```c
AVCodecParameters *pLocalCodecParameters = pFormatContext->streams[i]->codecpar;
```

With the codec properties we can look up the proper CODEC querying the function [`avcodec_find_decoder`](https://ffmpeg.org/doxygen/trunk/group__lavc__decoding.html#ga19a0ca553277f019dd5b0fec6e1f9dca) and find the registered decoder for the codec id and return an [`AVCodec`](http://ffmpeg.org/doxygen/trunk/structAVCodec.html), the component that knows how to en**CO**de and **DEC**ode the stream.
```c
AVCodec *pLocalCodec = avcodec_find_decoder(pLocalCodecParameters->codec_id);
```

Now we can print information about the codecs.

```c
// specific for video and audio
if (pLocalCodecParameters->codec_type == AVMEDIA_TYPE_VIDEO) {
  printf("Video Codec: resolution %d x %d", pLocalCodecParameters->width, pLocalCodecParameters->height);
} else if (pLocalCodecParameters->codec_type == AVMEDIA_TYPE_AUDIO) {
  printf("Audio Codec: %d channels, sample rate %d", pLocalCodecParameters->channels, pLocalCodecParameters->sample_rate);
}
// general
printf("\tCodec %s ID %d bit_rate %lld", pLocalCodec->long_name, pLocalCodec->id, pCodecParameters->bit_rate);
```

With the codec, we can allocate memory for the [`AVCodecContext`](https://ffmpeg.org/doxygen/trunk/structAVCodecContext.html), which will hold the context for our decode/encode process, but then we need to fill this codec context with CODEC parameters; we do that with [`avcodec_parameters_to_context`](https://ffmpeg.org/doxygen/trunk/group__lavc__core.html#gac7b282f51540ca7a99416a3ba6ee0d16).

Once we filled the codec context, we need to open the codec. We call the function [`avcodec_open2`](https://ffmpeg.org/doxygen/trunk/group__lavc__core.html#ga11f785a188d7d9df71621001465b0f1d) and then we can use it.

```c
AVCodecContext *pCodecContext = avcodec_alloc_context3(pCodec);
avcodec_parameters_to_context(pCodecContext, pCodecParameters);
avcodec_open2(pCodecContext, pCodec, NULL);
```

Now we're going to read the packets from the stream and decode them into frames but first, we need to allocate memory for both components, the [`AVPacket`](https://ffmpeg.org/doxygen/trunk/structAVPacket.html) and [`AVFrame`](https://ffmpeg.org/doxygen/trunk/structAVFrame.html).

```c
AVPacket *pPacket = av_packet_alloc();
AVFrame *pFrame = av_frame_alloc();
```

Let's feed our packets from the streams with the function [`av_read_frame`](https://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#ga4fdb3084415a82e3810de6ee60e46a61) while it has packets.

```c
while (av_read_frame(pFormatContext, pPacket) >= 0) {
  //...
}
```

Let's **send the raw data packet** (compressed frame) to the decoder, through the codec context, using the function [`avcodec_send_packet`](https://ffmpeg.org/doxygen/trunk/group__lavc__decoding.html#ga58bc4bf1e0ac59e27362597e467efff3).

```c
avcodec_send_packet(pCodecContext, pPacket);
```

And let's **receive the raw data frame** (uncompressed frame) from the decoder, through the same codec context, using the function [`avcodec_receive_frame`](https://ffmpeg.org/doxygen/trunk/group__lavc__decoding.html#ga11e6542c4e66d3028668788a1a74217c).

```c
avcodec_receive_frame(pCodecContext, pFrame);
```

We can print the frame number, the [PTS](https://en.wikipedia.org/wiki/Presentation_timestamp), DTS, [frame type](https://en.wikipedia.org/wiki/Video_compression_picture_types) and etc.

```c
printf(
    "Frame %c (%d) pts %d dts %d key_frame %d [coded_picture_number %d, display_picture_number %d]",
    av_get_picture_type_char(pFrame->pict_type),
    pCodecContext->frame_number,
    pFrame->pts,
    pFrame->pkt_dts,
    pFrame->key_frame,
    pFrame->coded_picture_number,
    pFrame->display_picture_number
);
```

Finally we can save our decoded frame into a [simple gray image](https://en.wikipedia.org/wiki/Netpbm_format#PGM_example). The process is very simple, we'll use the `pFrame->data` where the index is related to the [planes Y, Cb and Cr](https://en.wikipedia.org/wiki/YCbCr), we just picked `0` (Y) to save our gray image.

```c
save_gray_frame(pFrame->data[0], pFrame->linesize[0], pFrame->width, pFrame->height, frame_filename);

static void save_gray_frame(unsigned char *buf, int wrap, int xsize, int ysize, char *filename)
{
    FILE *f;
    int i;
    f = fopen(filename,"w");
    // writing the minimal required header for a pgm file format
    // portable graymap format -> https://en.wikipedia.org/wiki/Netpbm_format#PGM_example
    fprintf(f, "P5\n%d %d\n%d\n", xsize, ysize, 255);

    // writing line by line
    for (i = 0; i < ysize; i++)
        fwrite(buf + i * wrap, 1, xsize, f);
    fclose(f);
}
```

And voilà! Now we have a gray scale image with 2MB:

![saved frame](/img/generated_frame.png)

## Chapter 1 - syncing audio and video

> **Be the player** - a young JS developer writing a new MSE video player.

Before we move to [code a transcoding example](#chapter-2---transcoding) let's talk about **timing**, or how a video player knows the right time to play a frame.

In the last example, we saved some frames that can be seen here:

![frame 0](/img/hello_world_frames/frame0.png)
![frame 1](/img/hello_world_frames/frame1.png)
![frame 2](/img/hello_world_frames/frame2.png)
![frame 3](/img/hello_world_frames/frame3.png)
![frame 4](/img/hello_world_frames/frame4.png)
![frame 5](/img/hello_world_frames/frame5.png)

When we're designing a video player we need to **play each frame at a given pace**, otherwise it would be hard to pleasantly see the video either because it's playing so fast or so slow.

Therefore we need to introduce some logic to play each frame smoothly. For that matter, each frame has a **presentation timestamp** (PTS) which is an increasing number factored in a **timebase** that is a rational number (where the denominator is known as **timescale**) divisible by the **frame rate (fps)**.

It's easier to understand when we look at some examples, let's simulate some scenarios.

For a `fps=60/1` and `timebase=1/60000` each PTS will increase `timescale / fps = 1000` therefore the **PTS real time** for each frame could be (supposing it started at 0):

* `frame=0, PTS = 0, PTS_TIME = 0`
* `frame=1, PTS = 1000, PTS_TIME = PTS * timebase = 0.016`
* `frame=2, PTS = 2000, PTS_TIME = PTS * timebase = 0.033`

For almost the same scenario but with a timebase equal to `1/60`.

* `frame=0, PTS = 0, PTS_TIME = 0`
* `frame=1, PTS = 1, PTS_TIME = PTS * timebase = 0.016`
* `frame=2, PTS = 2, PTS_TIME = PTS * timebase = 0.033`
* `frame=3, PTS = 3, PTS_TIME = PTS * timebase = 0.050`

For a `fps=25/1` and `timebase=1/75` each PTS will increase `timescale / fps = 3` and the PTS time could be:

* `frame=0, PTS = 0, PTS_TIME = 0`
* `frame=1, PTS = 3, PTS_TIME = PTS * timebase = 0.04`
* `frame=2, PTS = 6, PTS_TIME = PTS * timebase = 0.08`
* `frame=3, PTS = 9, PTS_TIME = PTS * timebase = 0.12`
* ...
* `frame=24, PTS = 72, PTS_TIME = PTS * timebase = 0.96`
* ...
* `frame=4064, PTS = 12192, PTS_TIME = PTS * timebase = 162.56`

Now with the `pts_time` we can find a way to render this synched with audio `pts_time` or with a system clock. The FFmpeg libav provides these info through its API:

- fps = [`AVStream->avg_frame_rate`](https://ffmpeg.org/doxygen/trunk/structAVStream.html#a946e1e9b89eeeae4cab8a833b482c1ad)
- tbr = [`AVStream->r_frame_rate`](https://ffmpeg.org/doxygen/trunk/structAVStream.html#ad63fb11cc1415e278e09ddc676e8a1ad)
- tbn = [`AVStream->time_base`](https://ffmpeg.org/doxygen/trunk/structAVStream.html#a9db755451f14e2bf590d4b85d82b32e6)

Just out of curiosity, the frames we saved were sent in a DTS order (frames: 1,6,4,2,3,5) but played at a PTS order (frames: 1,2,3,4,5). Also, notice how cheap are B-Frames in comparison to P or I-Frames.

```
LOG: AVStream->r_frame_rate 60/1
LOG: AVStream->time_base 1/60000
...
LOG: Frame 1 (type=I, size=153797 bytes) pts 6000 key_frame 1 [DTS 0]
LOG: Frame 2 (type=B, size=8117 bytes) pts 7000 key_frame 0 [DTS 3]
LOG: Frame 3 (type=B, size=8226 bytes) pts 8000 key_frame 0 [DTS 4]
LOG: Frame 4 (type=B, size=17699 bytes) pts 9000 key_frame 0 [DTS 2]
LOG: Frame 5 (type=B, size=6253 bytes) pts 10000 key_frame 0 [DTS 5]
LOG: Frame 6 (type=P, size=34992 bytes) pts 11000 key_frame 0 [DTS 1]
```

## Chapter 2 - transcoding
