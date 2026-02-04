# mod_audio_stream

A FreeSWITCH module that streams L16 audio from a channel to a websocket endpoint. If websocket sends back responses (eg. JSON) it can be effectively used with ASR engines such as IBM Watson etc., or any other purpose you find applicable.

#### About

- The purpose of `mod_audio_stream` was to make a simple, less dependent but yet effective module to stream audio and receive responses from websocket server. It uses [ixwebsocket](https://machinezone.github.io/IXWebSocket/), c++ library for websocket protocol which is compiled as a static library.
- The original module was inspired by [mod_audio_fork](https://github.com/drachtio/drachtio-freeswitch-modules/tree/main/modules/mod_audio_fork).

## Installation

### Dependencies
This module uses the freeswitch library (under windows they are freeswitchcore.link and freeswitch.dll) [to be continued]

The other dependencies are injected via vcpkg
[if you don't have vcpkg, install it, like:
git clone https://github.com/microsoft/vcpkg.git C:\vcpkg
cd C:\vcpkg
.\bootstrap-vcpkg.bat]

[In the following c:\vcpkg is supposed as vcpkg dir]

The other dependencies:
speexdsp:
.\vcpkg.exe install speexdsp:x64-windows

ixwebsocket:
.\vcpkg.exe install ixwebsocket:x64-windows

mbedtls:
.\vcpkg install mbedtls:x64-windows


### To build the module, from the cloned repository directory:
```
Set up the project (if it is not set already):

mkdir build && cd build
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DCMAKE_TOOLCHAIN_FILE=<vcpkg dir>/scripts/buildsystems/vcpkg.cmake -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL -DFREESWITCH_INCLUDE_DIR="<fs dir>\src\include" -DFREESWITCH_CORE_LIB="<fs dir>\x64\Release\freeswitchcore.lib" -DSPEEXDSP_INCLUDE_DIR="<vcpkg dir>/installed/x64-windows/include" -DSPEEXDSP_LIB="<vcpkg dir>/installed/x64-windows/lib/speexdsp.lib" -DFREESWITCH_MODULES_DIR="<target dir>"

[In my case:
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL -DFREESWITCH_INCLUDE_DIR="d:\GitSecondary\fs_experiment_github\src\include" -DFREESWITCH_CORE_LIB="d:\GitSecondary\fs_experiment_github\x64\Release\freeswitchcore.lib" -DSPEEXDSP_INCLUDE_DIR="C:/vcpkg/installed/x64-windows/include" -DSPEEXDSP_LIB="C:/vcpkg/installed/x64-windows/lib/speexdsp.lib" -DFREESWITCH_MODULES_DIR="d:\GitSecondary\fs_experiment_github/mod"]


To compile and build, run:

cmake --build build --config Release --verbose



### TCP streaming

To stream to a TCP socket instead of a WS endpoint, you need to change `STREAM_TIME` to "TCP" in `mod_audio_stream.h`.

### Bufferization

Bufferization interval in milliseconds is configured on compile time in `audio_streamer_glue.cpp` (`BUFFERIZATION_INTERVAL_MS`).

### Channel variables
The following channel variables can be used to fine tune websocket connection and also configure mod_audio_stream logging:

| Variable               | Description                                         | Default |
|------------------------|-----------------------------------------------------|---------|
| STREAM_MESSAGE_DEFLATE | true or 1, disables per message deflate             | off     |
| STREAM_HEART_BEAT      | number of seconds, interval to send the heart beat  | off     |
| STREAM_SUPPRESS_LOG    | true or 1, suppresses printing to log               | off     |
| STREAM_BUFFER_SIZE     | buffer duration in milliseconds, divisible by 20    | 20      |
| STREAM_EXTRA_HEADERS   | JSON object for additional headers in string format | none    |

- Per message deflate compression option is enabled by default. It can lead to a very nice bandwidth savings. To disable it set the channel var to `true|1`.
- Heart beat, sent every xx seconds when there is no traffic to make sure that load balancers do not kill an idle connection.
- Suppress parameter is omitted by default(false). All the responses from websocket server will be printed to the log. Not to flood the log you can suppress it by setting the value to `true|1`. Events are fired still, it only affects printing to the log.
- `Buffer Size` actually represents a duration of audio chunk sent to websocket. If you want to send e.g. 100ms audio packets to your ws endpoint
you would set this variable to 100. If ommited, default packet size of 20ms will be sent as grabbed from the audio channel (which is default FreeSWITCH frame size)
- Extra headers should be a JSON object with key-value pairs representing additional HTTP headers. Each key should be a header name, and its corresponding value should be a string.
  ```json
  {
      "Header1": "Value1",
      "Header2": "Value2",
      "Header3": "Value3"
  }

## API

### Commands
The freeswitch module exposes the following API commands:

```
uuid_audio_stream <uuid> start <wss-url> <mix-type> <sampling-rate> <metadata>
```
Attaches a media bug and starts streaming audio (in L16 format) to the websocket server. FS default is 8k. If sampling-rate is other than 8k it will be resampled.
- `uuid` - Freeswitch channel unique id
- `wss-url` - websocket url `ws://` or `wss://`
- `mix-type` - choice of 
  - "mono" - single channel containing caller's audio
  - "mixed" - single channel containing both caller and callee audio
  - "stereo" - two channels with caller audio in one and callee audio in the other.
- `sampling-rate` - choice of
  - "8k" = 8000 Hz sample rate will be generated
  - "16k" = 16000 Hz sample rate will be generated
- `metadata` - (optional) a valid `utf-8` text to send. It will be sent the first before audio streaming starts.

```
uuid_audio_stream <uuid> send_text <metadata>
```
Sends a text to the websocket server. Requires a valid `utf-8` text.

```
uuid_audio_stream <uuid> stop <metadata>
```
Stops audio stream and closes websocket connection. If _metadata_ is provided it will be sent before the connection is closed.

```
uuid_audio_stream <uuid> pause
```
Pauses audio stream

```
uuid_audio_stream <uuid> resume
```
Resumes audio stream

## Events
Module will generate the following event types:
- `mod_audio_stream::json`
- `mod_audio_stream::connect`
- `mod_audio_stream::disconnect`
- `mod_audio_stream::error`
- `mod_audio_stream::play`

### response
Message received from websocket endpoint. Json expected, but it contains whatever the websocket server's response is.
#### Freeswitch event generated
**Name**: mod_audio_stream::json
**Body**: WebSocket server response

### connect
Successfully connected to websocket server.
#### Freeswitch event generated
**Name**: mod_audio_stream::connect
**Body**: JSON
```json
{
	"status": "connected"
}
```

### disconnect
Disconnected from websocket server.
#### Freeswitch event generated
**Name**: mod_audio_stream::disconnect
**Body**: JSON
```json
{
	"status": "disconnected",
	"message": {
		"code": 1000,
		"reason": "Normal closure"
	}
}
```
- code: `<int>`
- reason: `<string>`

### error
There is an error with the connection. Multiple fields will be available on the event to describe the error.
#### Freeswitch event generated
**Name**: mod_audio_stream::error
**Body**: JSON
```json
{
	"status": "error",
	"message": {
		"retries": 1,
		"error": "Expecting status 101 (Switching Protocol), got 403 status connecting to wss://localhost, HTTP Status line: HTTP/1.1 403 Forbidden\r\n",
		"wait_time": 100,
		"http_status": 403
	}
}
```
- retries: `<int>`, error: `<string>`, wait_time: `<int>`, http_status: `<int>`

### play
**Name**: mod_audio_stream::play
**Body**: JSON

Websocket server may return JSON object containing base64 encoded audio to be played by the user. To use this feature, response must follow the format:
```json
{
  "type": "streamAudio",
  "data": {
    "audioDataType": "raw",
    "sampleRate": 8000,
    "audioData": "base64 encoded audio"
  }
}
```
- audioDataType: `<raw|wav>`

Event generated by the module (subclass: _mod_audio_stream::play_) will be the same as the `data` element with the **file** added to it representing filePath:
```json
{
  "audioDataType": "raw",
  "sampleRate": 8000,
  "file": "/path/to/the/file"
}
```
If printing to the log is not suppressed, `response` printed to the console will look the same as the event. The original response containing base64 encoded audio is replaced because it can be quite huge.

All the files generated by this feature will reside at the temp directory and will be deleted when the session is closed.