# RASA Voice Assistant

A sophisticated voice-enabled conversational AI assistant built with RASA, designed to serve as a sales consultant for the BYD Sealion 7 in the UAE market. This assistant combines natural language processing, speech recognition, and text-to-speech capabilities to provide an interactive voice experience.

## 🚀 Features

### Core Capabilities
- **Voice Interface**: Real-time speech recognition and text-to-speech conversion
- **Conversational AI**: Powered by RASA framework with advanced NLU processing
- **LLM Integration**: Enhanced responses using the Claude (Anthropic) API for natural conversations
- **BYD Sealion 7 Expertise**: Specialized knowledge about the BYD Sealion 7 vehicle
- **Multi-modal Communication**: Supports both text and voice interactions

### Technical Features
- **Speech Recognition**: Google Speech Recognition API integration
- **Text-to-Speech**: ElevenLabs API for high-quality voice synthesis
- **Fallback Handling**: Intelligent fallback mechanisms for unrecognized inputs
- **Context Awareness**: Maintains conversation context for coherent dialogues
- **Intent Classification**: Advanced intent recognition with confidence scoring

## 🏗️ Architecture

```
RASA Voice Assistant
├── RASA Core (Conversation Management)
├── NLU Pipeline (Intent Recognition)
├── LLM Service (Claude API Integration)
├── Voice Interface (Speech I/O)
└── Custom Actions (Business Logic)
```

## 📋 Prerequisites

- **Python 3.10.** RASA 3.6 supports 3.8-3.10 only; on 3.11+ `pip install`
  fails outright. You do **not** have to downgrade your system Python -
  install 3.10 alongside it and point only this project's virtualenv at it
  (see below).
- About 3 GB of free disk. The virtualenv is ~2.7 GB, most of it TensorFlow.
- Microphone and speakers (for `voice_interface.py` only - the REST API works
  without them).
- Internet connection for speech recognition and TTS.

### API keys required
- **Anthropic API key** - the LLM responses. https://console.anthropic.com
- **ElevenLabs API key** - text-to-speech.

## 🛠️ Installation

### 1. Get Python 3.10 alongside your current Python

Installing 3.10 does not remove or downgrade any other version; the
virtualenv below pins this project to it and your system default is
untouched.

| Platform | Command |
|---|---|
| Windows | `winget install Python.Python.3.10` (or the python.org installer) |
| macOS | `brew install python@3.10` |
| Ubuntu/Debian | `sudo add-apt-repository ppa:deadsnakes/ppa && sudo apt install python3.10 python3.10-venv` |
| Any (pyenv) | `pyenv install 3.10.20 && pyenv local 3.10.20` |
| Any (conda) | `conda create -n rasa python=3.10 && conda activate rasa` (then skip step 2) |

### 2. Create the virtualenv with that interpreter

```bash
python3.10 -m venv rasa_env310            # Windows: py -3.10 -m venv rasa_env310
source rasa_env310/bin/activate           # Windows: rasa_env310\Scripts\activate
python -V                                 # MUST print 3.10.x
```

If `python -V` reports anything else, the venv was built from the wrong
interpreter - delete `rasa_env310/` and redo it with the explicit
`python3.10` / `py -3.10` form. This is the most common setup failure.

### 3. Install dependencies

```bash
pip install -r requirements.txt
pip check                                 # expect: No broken requirements found
```

Takes several minutes (TensorFlow alone is a 586 MB download).

> Do not add `mcp` to `requirements.txt`. RASA 3.6 pins `pydantic<1.10.10`
> and every `mcp` release needs `pydantic>=2`, so the two cannot coexist -
> adding it makes the file unsatisfiable. An MCP server would need its own
> process and its own virtualenv.

### 4. Set up environment variables

```bash
cp .env.example .env
```

Then fill in both keys. Note the ElevenLabs variable name - the code reads
`ELEVEN_LABS_API2`, not `ELEVENLABS_API_KEY`:

```env
ANTHROPIC_API_KEY=your_anthropic_api_key_here
ELEVEN_LABS_API2=your_elevenlabs_api_key_here
# Optional - defaults to claude-opus-5
# ANTHROPIC_MODEL=claude-opus-5
```

## 🚀 Quick Start

Activate the virtualenv in every terminal you open (`source
rasa_env310/bin/activate`).

### 1. Train the RASA model
```bash
rasa train
```
Takes roughly 2 minutes and ends with `Your Rasa model is trained and saved
at 'models/<timestamp>-<name>.tar.gz'`. Required on a fresh clone - no
trained model ships in the repo, and `models/*.tar.gz` is gitignored.

### 2. Start the action server (terminal 1, start this first)
The custom actions in `actions/actions.py` are what call Claude, so this
must be running or every LLM response falls back to an error message.
```bash
rasa run actions
```
Wait for `Action endpoint is up and running on http://0.0.0.0:5055`, which
matches `action_endpoint` in `endpoints.yml`.

### 3. Start the RASA server (terminal 2)
```bash
rasa run --enable-api --cors "*"
```
This sits on `Loading model models/...` for about **75 seconds** while
TensorFlow loads. That is normal, not a hang.

### 4. Check it by text before involving the microphone

This isolates the RASA + Claude path from all the audio variables:

```bash
curl -s -X POST http://localhost:5005/webhooks/rest/webhook \
  -H "Content-Type: application/json" \
  -d '{"sender":"me","message":"tell me about the performance of this car"}'
```

You should get a JSON reply. Terminal 1 will log the token usage:

```
tokens: uncached=66  cache_write=2253  cache_read=0  output=20
```

Ask a second question and it should read `cache_write=0 cache_read=2253` -
the knowledge base is cached after the first call. If a later turn logs
another `cache_write`, or you see `Prompt cache did not engage`, something
is varying the cached prompt prefix and the knowledge base is being
re-billed every turn.

### 5. Start the voice interface (terminal 3)

This step - and only this step - needs **PyAudio**, which
`speech_recognition` uses to open the microphone. It is deliberately not in
`requirements.txt`: on Linux and macOS it builds from source against
portaudio, so listing it would break `pip install -r requirements.txt` for
anyone not doing voice.

| Platform | Command |
|---|---|
| Windows | `pip install pyaudio` (prebuilt wheels, nothing else needed) |
| macOS | `brew install portaudio && pip install pyaudio` |
| Ubuntu/Debian | `sudo apt install portaudio19-dev && pip install pyaudio` |

Without it you get `AttributeError: Could not find PyAudio; check
installation` the moment `VoiceAgent` is constructed.

```bash
python voice_interface.py
```
It calibrates the microphone for ambient noise, speaks its greeting, then
listens. Say "goodbye" to exit.

## 📁 Project Structure

```
RASA_Voice_Assistant/
├── actions/                 # Custom RASA actions
│   ├── __init__.py
│   ├── actions.py          # LLM integration actions
│   └── knowledge_base.txt  # BYD Sealion 7 information
├── data/                   # Training data
│   ├── nlu.yml            # Intent and entity training
│   ├── stories.yml        # Conversation flows
│   └── rules.yml          # Business rules
├── models/                 # Trained RASA models (produced by `rasa train`, not committed)
├── config.yml             # RASA configuration
├── domain.yml             # Domain definition
├── credentials.yml        # API credentials
├── endpoints.yml          # Service endpoints
├── voice_interface.py    # Voice interface implementation
├── llm_service.py        # LLM service integration
├── requirements.txt      # Python dependencies
├── .env.example          # Template for your .env file
└── README.md            # This file
```

## 🎯 Usage Examples

### Voice Interaction
1. **Start the assistant**: "Hello" or "Hi"
2. **Ask about BYD Sealion 7**: "Tell me about the Sealion 7"
3. **Performance questions**: "What's the acceleration like?"
4. **Exit**: "Goodbye" or "Bye"

### Supported Intents
- `greet`: Greeting messages
- `goodbye`: Farewell messages
- `ask_question`: General questions
- `ask_weather`: Weather-related queries
- `bot_challenge`: Questions about the bot
- `nlu_fallback`: Unrecognized inputs

## 🔧 Configuration

### RASA Configuration (`config.yml`)
- **Pipeline**: DIET classifier with 100 epochs
- **Policies**: Memoization, Rule, UnexpecTEDIntent, and TED policies
- **Fallback**: 0.3 threshold with ambiguity detection

### Voice Settings
- **Speech Rate**: 180 words per minute
- **Voice**: Female voice (configurable)
- **Audio Quality**: High-quality ElevenLabs synthesis

### LLM Configuration
- **Provider**: Claude API (Anthropic Python SDK)
- **Model**: `claude-opus-5` (override with `ANTHROPIC_MODEL`; `claude-haiku-4-5` is a cheaper, lower-latency option for live calls)
- **Effort**: `low` - these are short conversational turns, so shallow thinking keeps latency down
- **Max Tokens**: 2048 - the reply is capped at 40 words by the prompt, but adaptive thinking draws from the same budget
- **Prompt caching**: the Rachel persona is sent as a cached system block, so repeat turns in a call are billed at ~0.1x for that prefix

## 🧪 Testing

### Test Stories
```bash
rasa test --stories tests/test_stories.yml
```

### Interactive Testing
```bash
rasa shell
```

### Voice Testing
```bash
python voice_interface.py
```

## 🔍 Troubleshooting

### Setup failures (in order of likelihood)

**`pip install` fails with `ResolutionImpossible`.** Either the virtualenv was
built with Python 3.11+, or `mcp` was added back to `requirements.txt`. Check
`python -V` inside the activated venv first - it must say 3.10.x.

**Every reply is "I'm sorry, I couldn't process that request right now."**
The action server (terminal 1) is not running or has crashed. This fails
*silently*: RASA returns the fallback string rather than an error, so nothing
looks broken from the client side. Check terminal 1.

**The RASA server looks frozen on `Loading model ...`.** Give it ~75 seconds.
It is loading TensorFlow, not hung.

**Replies work but nothing is spoken, and the log says "ElevenLabs API key not
provided!".** The key is missing or under the wrong name in `.env`. The code
reads `ELEVEN_LABS_API2`.

**`Prompt cache did not engage`, or every turn logs `cache_write` instead of
`cache_read`.** Something is varying the cached prompt prefix between turns,
so the knowledge base is re-sent at full price on every turn. The prefix
(persona + knowledge base) is built once in `LLMService.__init__` and must be
byte-identical across requests.

### Common Issues

1. **Speech Recognition Not Working**
   - Check microphone permissions
   - Ensure internet connection for Google Speech API
   - Verify microphone is properly connected

2. **TTS Not Working**
   - Verify ElevenLabs API key in `.env` file
   - Check internet connection
   - Ensure audio output is working

3. **RASA Server Issues**
   - Check if port 5005 is available
   - Verify model training completed successfully
   - Check logs for specific error messages

4. **LLM Service Errors**
   - Verify Anthropic API key
   - Check internet connection
   - Ensure API quota is not exceeded

### Debug Mode
Enable debug logging by setting:
```python
logging.basicConfig(level=logging.DEBUG)
```

## 📊 Performance

### Model Performance
- **Training Time**: ~5-10 minutes (depending on hardware)
- **Response Time**: <2 seconds for voice interactions
- **Accuracy**: High intent recognition with fallback handling

### Voice Quality
- **ElevenLabs**: High-quality, natural-sounding speech
- **Latency**: Optimized for real-time conversation
- **Customization**: Configurable voice settings

## 🔒 Security

- API keys stored in environment variables
- No sensitive data in code repository
- Secure HTTPS communication for APIs
- Local speech processing where possible

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request


## 🙏 Acknowledgments

- **RASA**: Open-source conversational AI framework
- **ElevenLabs**: High-quality text-to-speech API
- **Anthropic**: Claude API for LLM responses
- **Google Speech Recognition**: Speech-to-text capabilities

## 📞 Support

For support and questions:
- Create an issue in the repository
- Check the troubleshooting section
- Review RASA documentation

---
