![mini_al](http://dred.io/img/minial_wide.png)

mini_al is a single file library for playing and recording audio. It's written in C (compilable as C++)
and released into the public domain. It has a heavy focus on simplicity, and features a simple API and
a very simple build system.


Features
========
- Public domain.
- Single file.
- Compilable as both C and C++.
- Easy to build.
  - Does not require linking to anything on the Windows build and only -ldl on Linux.
  - It should Just Work out of the box, without the need to download and install any dependencies. (Note
    that some older versions of MinGW/MinGW-64 don't include WASAPI headers which may require manual
    installation. Plans are in place to work around this in a future update.)
  - The header section does not include any platform specific headers.
- A very simple API.
- Transparent data structures with direct access to internal data.
- Supports both playback and capture on all backends.
- Automatic format conversion.
  - Sample format conversion.
  - Sample rate conversion.
    - Sample rate conversion is currently low quality, but a higher quality implementation is planned.
  - Channel mapping/layout.
  - Channel mixing (converting mono to 5.1, etc.)
- MP3, Vorbis, FLAC and WAV decoding.
  - This depends on external single file libraries which can be found in the "extras" folder.


Supported Platforms
===================
- Windows (XP+)
- Linux
- BSD
- Android
- Emscripten / HTML5

macOS and iOS support is coming soon(ish) via Core Audio. Unofficial support is enabled via the
PulseAudio, JACK, OpenAL and SDL backends, however I have not tested these personally.


Backends
========
- WASAPI
- DirectSound
- WinMM
- ALSA
- PulseAudio
- JACK
- OSS
- OpenSL|ES (Android only)
- OpenAL
- SDL
- Null (Silence)



Simple Playback Example
=======================

```c
#define DR_FLAC_IMPLEMENTATION
#include "../extras/dr_flac.h"  // Enables FLAC decoding.
#define DR_MP3_IMPLEMENTATION
#include "../extras/dr_mp3.h"   // Enables MP3 decoding.
#define DR_WAV_IMPLEMENTATION
#include "../extras/dr_wav.h"   // Enables WAV decoding.

#define MAL_IMPLEMENTATION
#include "../mini_al.h"

#include <stdio.h>

// This is the function that's used for sending more data to the device for playback.
mal_uint32 on_send_frames_to_device(mal_device* pDevice, mal_uint32 frameCount, void* pSamples)
{
    mal_decoder* pDecoder = (mal_decoder*)pDevice->pUserData;
    if (pDecoder == NULL) {
        return 0;
    }

    return (mal_uint32)mal_decoder_read(pDecoder, frameCount, pSamples);
}

int main(int argc, char** argv)
{
    if (argc < 2) {
        printf("No input file.\n");
        return -1;
    }

    mal_decoder decoder;
    mal_result result = mal_decoder_init_file(argv[1], NULL, &decoder);
    if (result != MAL_SUCCESS) {
        return -2;
    }

    mal_device_config config = mal_device_config_init_playback(
        decoder.outputFormat,
        decoder.outputChannels,
        decoder.outputSampleRate,
        on_send_frames_to_device);

    mal_device device;
    if (mal_device_init(NULL, mal_device_type_playback, NULL, &config, &decoder, &device) != MAL_SUCCESS) {
        printf("Failed to open playback device.\n");
        mal_decoder_uninit(&decoder);
        return -3;
    }

    if (mal_device_start(&device) != MAL_SUCCESS) {
        printf("Failed to start playback device.\n");
        mal_device_uninit(&device);
        mal_decoder_uninit(&decoder);
        return -4;
    }

    printf("Press Enter to quit...");
    getchar();

    mal_device_uninit(&device);
    mal_decoder_uninit(&decoder);

    return 0;
}
```


MP3/Vorbis/FLAC/WAV Decoding
============================
mini_al includes a decoding API which supports the following backends:
- FLAC via [dr_flac](https://github.com/mackron/dr_libs/blob/master/dr_flac.h)
- MP3 via [dr_mp3](https://github.com/mackron/dr_libs/blob/master/dr_mp3.h)
- WAV via [dr_wav](https://github.com/mackron/dr_libs/blob/master/dr_wav.h)
- Vorbis via [stb_vorbis](https://github.com/nothings/stb/blob/master/stb_vorbis.c)

Copies of these libraries can be found in the "extras" folder. You may also want to look at the
libraries below, but they are not supported by the mini_al decoder API. If you know of any other
single file libraries I can add to this list, let me know. Preferably public domain or MIT.
- [minimp3](https://github.com/lieff/minimp3)
- [jar_mod](https://github.com/kd7tck/jar/blob/master/jar_mod.h)
- [jar_xm](https://github.com/kd7tck/jar/blob/master/jar_xm.h)

To enable support for a decoding backend, all you need to do is #include the header section of the
relevant backend library before the implementation of mini_al, like so:

```
#include "dr_flac.h"    // Enables FLAC decoding.
#include "dr_mp3.h"     // Enables MP3 decoding.
#include "dr_wav.h"     // Enables WAV decoding.

#define MAL_IMPLEMENTATION
#include "mini_al.h"
```

A decoder can be initialized from a file with `mal_decoder_init_file()`, a block of memory with
`mal_decoder_init_memory()`, or from data delivered via callbacks with `mal_decoder_init()`. Here
is an example for loading a decoder from a file:

```
mal_decoder decoder;
mal_result result = mal_decoder_init_file("MySong.mp3", NULL, &decoder);
if (result != MAL_SUCCESS) {
    return false;   // An error occurred.
}

...

mal_decoder_uninit(&decoder);
```

When initializing a decoder, you can optionally pass in a pointer to a `mal_decoder_config` object
(the `NULL` argument in the example above) which allows you to configure the output format, channel
count, sample rate and channel map:

```
mal_decoder_config config = mal_decoder_config_init(mal_format_f32, 2, 48000);
```

When passing in NULL for this parameter, the output format will be the same as that defined by the
decoding backend.

Data is read from the decoder as PCM frames:

```
mal_uint64 framesRead = mal_decoder_read(pDecoder, framesToRead, pFrames);
```

You can also seek to a specific frame like so:

```
mal_result result = mal_decoder_seek(pDecoder, targetFrame);
if (result != MAL_SUCCESS) {
    return false;   // An error occurred.
}
```

When loading a decoder, mini_al uses a trial and error technique to find the appropriate decoding
backend. This can be unnecessarily inefficient if the type is already known. In this case you can
use the `_wav`, `_mp3`, etc. varients of the aforementioned initialization APIs:

```
mal_decoder_init_wav()
mal_decoder_init_mp3()
mal_decoder_init_memory_wav()
mal_decoder_init_memory_mp3()
mal_decoder_init_file_wav()
mal_decoder_init_file_mp3()
etc.
```

The `mal_decoder_init_file()` API will try using the file extension to determine which decoding
backend to prefer.