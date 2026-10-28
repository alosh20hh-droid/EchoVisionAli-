# EchoVision with OpenRouter Integration

## 🎯 Overview

This is an enhanced version of EchoVision that integrates with **OpenRouter** - a universal API gateway for accessing multiple LLM providers (OpenAI, Anthropic Claude, Meta Llama, etc.) through a single interface.

### Why OpenRouter?

✅ **Cost-effective** - Compare prices across providers  
✅ **Model flexibility** - Switch between models easily  
✅ **Multiple providers** - OpenAI, Claude, Llama, Mistral, etc.  
✅ **Fallback support** - Automatic failover between models  
✅ **Usage monitoring** - Dashboard to track spending  

---

## 🚀 Quick Start

### 1. Get Your API Key

1. Go to https://openrouter.io/
2. Sign up (free account)
3. Navigate to https://openrouter.io/keys
4. Copy your API key

### 2. Setup Environment

```bash
# Clone/enter the project
cd EchoVision-OpenRouter/host

# Copy example .env file
cp .env.example .env

# Edit .env with your API key
# OPENROUTER_API_KEY=your_key_here
nano .env
```

### 3. Install Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install requirements
pip install -r ../requirements.txt
```

### 4. Test the Integration

```bash
# Test with a sample image
python test_chain.py

# Expected output:
# - Vision analysis of the image
# - Action chain recommendations
# - API response from OpenRouter
```

---

## 📋 Configuration

### Using OpenRouter (Default)

```python
from reasoning import reasoning

# Automatically uses OpenRouter with GPT-4o Mini
response = reasoning(
    images=['base64_encoded_image'],
    query="What do you see?",
    provider='openrouter'
)
```

### Using OpenAI Directly

```python
# Set OPENAI_API_KEY in .env, then:
response = reasoning(
    images=['base64_encoded_image'],
    query="What do you see?",
    provider='openai'
)
```

### Using Different Models

OpenRouter supports dozens of models. Here are some popular options:

**Fast & Cheap:**
```python
reasoning(images, query, provider='openrouter', model='openai/gpt-4o-mini')
reasoning(images, query, provider='openrouter', model='anthropic/claude-3-haiku')
reasoning(images, query, provider='openrouter', model='meta-llama/llama-2-70b-chat')
```

**Most Capable:**
```python
reasoning(images, query, provider='openrouter', model='openai/gpt-4')
reasoning(images, query, provider='openrouter', model='anthropic/claude-3-opus')
```

See all models at: https://openrouter.io/models

---

## 🔧 Architecture Changes

### Enhanced reasoning.py

**Old (OpenAI-only):**
```python
from openai import OpenAI

client = OpenAI(api_key=...)
response = client.chat.completions.create(...)
```

**New (Multi-provider):**
```python
from reasoning import LLMConfig, reasoning

config = LLMConfig(provider='openrouter')
response = reasoning(images, query, provider='openrouter')
```

### Key Features

1. **LLMConfig Class** - Centralized configuration management
   - Stores provider details
   - Manages API keys
   - Validates configuration
   - Handles model selection

2. **Flexible reasoning() Function**
   - Supports multiple providers
   - Automatic API key detection
   - Error handling and logging
   - Request timeout management
   - Vision image formatting

3. **Provider Abstraction**
   - Consistent API interface
   - Easy provider switching
   - Fallback support ready

---

## 📊 Project Structure

```
EchoVision-OpenRouter/
├── host/
│   ├── reasoning.py          ← ENHANCED: Multi-provider LLM
│   ├── test_chain.py         ← Action chain testing
│   ├── action.py             ← Action execution
│   ├── main.py               ← Camera & agent processes
│   ├── buffer.py             ← Frame buffering
│   ├── transcription.py       ← Speech-to-text
│   ├── tts.py                ← Text-to-speech
│   ├── .env.example          ← UPDATED: Configuration template
│   └── Dockerfile
├── embedded/                 ← Raspberry Pi embedded code
├── container/                ← Docker configuration
├── requirements.txt          ← UPDATED: Dependencies
└── SETUP_OPENROUTER.md      ← THIS FILE
```

---

## 🎯 Action Chain System

The system uses intelligent action chaining to solve complex tasks:

### Example Flow

**User Query:** "Give me directions to the nearest restaurant"

```
1. Call reasoning() with images + query
   ↓
2. Model responds with action chain:
   "Action Chain: get location → find restaurants → get directions"
   "Next Action: get_current_location()"
   ↓
3. Execute action, get result: "37.7749, -122.4194"
   ↓
4. Call reasoning() again with result as param
   ↓
5. Model responds: "Next Action: get_nearby_places(37.7749, -122.4194, 'restaurant')"
   ↓
6. Execute, get: "Luigi's Restaurant, 123 Main St"
   ↓
7. Call reasoning() again
   ↓
8. Model responds: "Next Action: get_route_to_destination(37.7749, -122.4194, Luigi's)"
   ↓
9. Execute, return final directions to user
```

---

## 🔄 Integration with visionpod

To integrate this with your visionpod project:

1. **Replace reasoning.py** in visionpod with the enhanced version
2. **Update requirements.txt** to include new dependencies
3. **Copy .env.example** as template
4. **Update app.py** to use new LLMConfig:

```python
from reasoning import reasoning, LLMConfig

# Initialize once
config = LLMConfig(provider='openrouter')

# Use in your routes
@app.post("/analyze")
async def analyze(image: UploadFile, question: str):
    response = reasoning(
        images=[base64_image],
        query=question,
        provider='openrouter'
    )
    return {"answer": response}
```

---

## 🐛 Troubleshooting

### "Missing API key" Error

```
ValueError: Missing API key. Set OPENROUTER_API_KEY environment variable
```

**Solution:**
- Create `.env` file in the host/ directory
- Add your OpenRouter API key: `OPENROUTER_API_KEY=your_key_here`
- Ensure .env is NOT in .gitignore (or it's ignored)

### "Request timeout" Error

```
requests.exceptions.Timeout: ...
```

**Solution:**
- OpenRouter might be slow for first request
- Increase timeout: `reasoning(images, query, timeout=60)`
- Try a faster model: `model='openai/gpt-4o-mini'`

### "403 Unauthorized" Error

```
Error calling openrouter API: 403 Forbidden
```

**Solution:**
- Check API key is correct
- Regenerate key at https://openrouter.io/keys
- Ensure account has credits

### Images Not Processed

```
Error processing image: ...
```

**Solution:**
- Ensure images are base64 encoded
- Check image format (JPEG/PNG)
- Verify file size < 20MB

---

## 📈 Monitoring & Costs

### Using OpenRouter Dashboard

1. Visit https://openrouter.io/activity
2. Monitor API usage
3. View costs per model
4. Set spending limits

### Cost Comparison (for 1000 image+text requests)

| Model | Price |
|-------|-------|
| GPT-4o Mini | $0.30-0.50 |
| Claude 3 Haiku | $0.10-0.20 |
| Llama 2 70B | Free |
| GPT-4 | $5-10 |

---

## 🚀 Production Deployment

### Environment Variables

```bash
# For production, set these env vars instead of .env file:
export OPENROUTER_API_KEY=your_key
export LLM_PROVIDER=openrouter
export REQUEST_TIMEOUT=30
```

### Docker Deployment

```bash
# Build with OpenRouter support
docker build -t echovision-openrouter .

# Run with env vars
docker run \
  -e OPENROUTER_API_KEY=your_key \
  -p 8000:8000 \
  echovision-openrouter
```

### Logging

Enable debug logging:
```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

---

## 🔗 Useful Links

- **OpenRouter**: https://openrouter.io/
- **Available Models**: https://openrouter.io/models
- **API Docs**: https://openrouter.io/docs/quick-start
- **Pricing**: https://openrouter.io/#pricing

---

## 📝 Next Steps

1. ✅ Get OpenRouter API key
2. ✅ Set up .env file
3. ✅ Test with test_chain.py
4. ✅ Integrate with visionpod
5. ✅ Configure for production

---

## 💡 Tips & Best Practices

### 1. Cost Optimization

```python
# Use cheaper models for simple tasks
reasoning(images, "Describe the image", model='anthropic/claude-3-haiku')

# Use premium models for complex reasoning
reasoning(images, "Complex analysis task", model='openai/gpt-4')
```

### 2. Error Handling

```python
try:
    response = reasoning(images, query)
except Exception as e:
    logger.error(f"API Error: {e}")
    # Fallback to cached response or simpler model
    response = reasoning(images, query, model='openai/gpt-4o-mini')
```

### 3. Performance Optimization

```python
# Cache images to avoid re-encoding
base64_images = [get_image_base64(path) for path in image_paths]
response = reasoning(base64_images, query)
```

---

**Last Updated:** October 2026  
**Status:** Production Ready ✅
