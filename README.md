# OpenedAI Speech

An OpenAI API compatible text to speech server.

* Compatible with the OpenAI audio/speech API
* Serves the [/v1/audio/speech endpoint](https://platform.openai.com/docs/api-reference/audio/createSpeech)
* Not affiliated with OpenAI in any way, does not require an OpenAI API Key
* A free, private, text-to-speech server with custom voice cloning

Full Compatibility:
* `tts-1`: `alloy`, `echo`, `fable`, `onyx`, `nova`, and `shimmer` (configurable)
* `tts-1-hd`:  `alloy`, `echo`, `fable`, `onyx`, `nova`, and `shimmer` (configurable, uses OpenAI samples by default)
* response_format: `mp3`, `opus`, `aac`, or `flac`
* speed 0.25-4.0 (and more)

Details:
* model 'tts-1' via [piper tts](https://github.com/rhasspy/piper) (fast, can use cpu)
* model 'tts-1-hd' via [coqui-ai/TTS](https://github.com/coqui-ai/TTS) xtts_v2 voice cloning (fast, but requires around 4GB GPU VRAM)
* Can be run without TTS/xtts_v2, entirely on cpu
* Custom cloned voices can be used for tts-1-hd, just save a WAV file in the `/voices/` directory
* You can map your own [piper voices](https://rhasspy.github.io/piper-samples/) and xtts_v2 speaker clones via the `voice_to_speaker.yaml` configuration file
* Occasionally, certain words or symbols may sound incorrect, you can fix them with regex via `pre_process_map.yaml`

If you find a better voice match for `tts-1` or `tts-1-hd`, please let me know so I can update the defaults.

## Recent Changes

Version: 0.10.1, 2024-05-05

* Remove `runtime: nvidia` from docker-compose.yml, this assumes nvidia/cuda compatible runtime is available by default. thanks @jmtatsch

Version: 0.10.0, 2024-04-27

* Pre-built & tested docker images, smaller docker images (8GB or 860MB)
* Better upgrades: reorganize config files under `config/`, voice models under `voices/`
* **Compatibility!** If you customized your `voice_to_speaker.yaml` or `pre_process_map.yaml` you need to move them to the `config/` folder.
* default listen host to 0.0.0.0

Version: 0.9.0, 2024-04-23

* Fix bug with yaml and loading UTF-8
* New sample text-to-speech application `say.py`
* Smaller docker base image
* Add beta [parler-tts](https://huggingface.co/parler-tts/parler_tts_mini_v0.1) support (you can describe very basic features of the speaker voice), See: (https://www.text-description-to-speech.com/) for some examples of how to describe voices. Voices can be defined in the `voice_to_speaker.default.yaml`. Two example [parler-tts](https://huggingface.co/parler-tts/parler_tts_mini_v0.1) voices are included in the `voice_to_speaker.default.yaml` file. `parler-tts` is experimental software and is kind of slow. The exact voice will be slightly different each generation but should be similar to the basic description.

...

Version: 0.7.3, 2024-03-20

* Allow different xtts versions per voice in `voice_to_speaker.yaml`, ex. xtts_v2.0.2
* Quality: Fix xtts sample rate (24000 vs. 22050 for piper) and pops


## Installation instructions

1) Download the models & voices
```shell
# for tts-1 / piper
bash download_voices_tts-1.sh
# and for tts-1-hd / xtts
bash download_voices_tts-1-hd.sh
```

If you have different models which you want to use, both of the download scripts accept arguments for which models to download.

Example:
```shell
# Download en_US-ryan-high too
bash download_voices_tts-1.sh en_US-libritts_r-medium en_GB-northern_english_male-medium en_US-ryan-high
# Download xtts (latest) and xtts_v2.0.2
bash download_voices_tts-1-hd.sh xtts xtts_v2.0.2
```


2a) Option 1: Docker (**recommended**) (prebuilt images are available)

You can run the server via docker like so:
```shell
cp sample.env speech.env # edit to suit your environment as needed, you can preload a model on startup
docker compose up
```
If you want a minimal docker image with piper support only (<1GB vs. 8GB, see: Dockerfile.min). You can edit the `docker-compose.yml` to easily change this.
To install the docker image as a service, edit the `docker-compose.yml` and uncomment `restart: unless-stopped`, then start the service with: `docker compose up -d`.


2b) Option 2: Manual instructions:
```shell
# install ffmpeg and curl
sudo apt install ffmpeg curl
# Create & activate a new virtual environment
python -m venv .venv
source .venv/bin/activate
# Install the Python requirements
pip install -r requirements.txt
# run the server
python speech.py
```

## Usage

```
usage: speech.py [-h] [--piper_cuda] [--xtts_device XTTS_DEVICE] [--preload PRELOAD] [-P PORT] [-H HOST]

OpenedAI Speech API Server

options:
  -h, --help            show this help message and exit
  --piper_cuda          Enable cuda for piper. Note: --cuda/onnxruntime-gpu is not working for me, but cpu is fast enough (default: False)
  --xtts_device XTTS_DEVICE
                        Set the device for the xtts model. The special value of 'none' will use piper for all models. (default: cuda)
  --preload PRELOAD     Preload a model (Ex. 'xtts' or 'xtts_v2.0.2'). By default it's loaded on first use. (default: None)
  -P PORT, --port PORT  Server tcp port (default: 8000)
  -H HOST, --host HOST  Host to listen on, Ex. 0.0.0.0 (default: 0.0.0.0)

```

## API Documentation

* [OpenAI Text to speech guide](https://platform.openai.com/docs/guides/text-to-speech)
* [OpenAI API Reference](https://platform.openai.com/docs/api-reference/audio/createSpeech)


### Sample API Usage

You can use it like this:

```shell
curl http://localhost:8000/v1/audio/speech -H "Content-Type: application/json" -d '{
    "model": "tts-1",
    "input": "The quick brown fox jumped over the lazy dog.",
    "voice": "alloy",
    "response_format": "mp3",
    "speed": 1.0
  }' > speech.mp3
```

Or just like this:

```shell
curl http://localhost:8000/v1/audio/speech -H "Content-Type: application/json" -d '{
    "input": "The quick brown fox jumped over the lazy dog."}' > speech.mp3
```

Or like this example from the [OpenAI Text to speech guide](https://platform.openai.com/docs/guides/text-to-speech):

```python
import openai

client = openai.OpenAI(
  # This part is not needed if you set these environment variables before import openai
  # export OPENAI_API_KEY=sk-11111111111
  # export OPENAI_BASE_URL=http://localhost:8000/v1
  api_key = "sk-111111111",
  base_url = "http://localhost:8000/v1",
)

with client.audio.speech.with_streaming_response.create(
  model="tts-1",
  voice="alloy",
  input="Today is a wonderful day to build something people love!"
) as response:
  response.stream_to_file("speech.mp3")
```

Also see the `say.py` sample application for an example of how to use the openai-python API.

```shell
python say.py -t "The quick brown fox jumped over the lazy dog." -p # play the audio, requires 'pip install playsound'
python say.py -t "The quick brown fox jumped over the lazy dog." -m tts-1-hd -v onyx -f flac -o fox.flac # save to a file.
```

```
usage: say.py [-h] [-m MODEL] [-v VOICE] [-f {mp3,aac,opus,flac}] [-s SPEED] [-t TEXT] [-i INPUT] [-o OUTPUT] [-p]

Text to speech using the OpenAI API

options:
  -h, --help            show this help message and exit
  -m MODEL, --model MODEL
                        The model to use (default: tts-1)
  -v VOICE, --voice VOICE
                        The voice of the speaker (default: alloy)
  -f {mp3,aac,opus,flac}, --format {mp3,aac,opus,flac}
                        The output audio format (default: mp3)
  -s SPEED, --speed SPEED
                        playback speed, 0.25-4.0 (default: 1.0)
  -t TEXT, --text TEXT  Provide text to read on the command line (default: None)
  -i INPUT, --input INPUT
                        Read text from a file (default is to read from stdin) (default: None)
  -o OUTPUT, --output OUTPUT
                        The filename to save the output to (default: None)
  -p, --playsound       Play the audio (default: False)
```

## Custom Voices Howto

Custom voices should be mono 22050 hz sample rate WAV files with low noise (no background music, etc.) and not contain any partial words.Sample voices for xtts should be at least 6 seconds long, but they can be longer. However, longer samples do not always produce better results.

You can use FFmpeg to process your audio files and prepare them for xtts, here are some examples:

```shell
# convert a multi-channel audio file to mono, set sample rate to 22050 hz, trim to 6 seconds, and output as WAV file.
ffmpeg -i input.mp3 -ac 1 -ar 22050 -t 6 -y me.wav
# use a simple noise filter to clean up audio, and select a start time start for sampling.
ffmpeg -i input.wav -af "highpass=f=200, lowpass=f=3000" -ac 1 -ar 22050 -ss 00:13:26.2 -t 6 -y me.wav
# A more complex noise reduction setup, including volume adjustment
ffmpeg -i input.mkv -af "highpass=f=200, lowpass=f=3000, volume=5, afftdn=nf=25" -ac 1 -ar 22050 -ss 00:13:26.2 -t 6 -y me.wav
```

Once your WAV file is prepared, save it in the `/voices/` directory and update the `voice_to_speaker.yaml` file with the new file name.

For example:

```yaml
...
tts-1-hd:
  me:
    model: xtts_v2.0.2 # you can specify different xtts versions
    speaker: voices/me.wav # this could be you
```
