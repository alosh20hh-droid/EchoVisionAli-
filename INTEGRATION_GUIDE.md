# Integration Guide: OpenRouter with visionpod

## 🎯 Goal

Integrate OpenRouter support into your **visionpod** project with professional-grade features:
- Multiple LLM providers
- Advanced error handling
- Logging and monitoring
- Rate limiting
- Request timeout management

---

## 📊 Step-by-Step Integration (Applies to visionpod)

### Phase 1: Setup & Dependencies

#### 1.1 Update requirements.txt

Add to your visionpod `requirements.txt`:

```txt
# LLM APIs
requests==2.31.0
python-dotenv==1.0.1

# Optional but recommended
colorama==0.4.6
tenacity==8.2.3  # For retry logic
```

#### 1.2 Create Configuration Module

Create `config.py` in your project root:

```python
"""Configuration management for visionpod with multi-provider LLM support"""

import os
from dataclasses import dataclass
from dotenv import load_dotenv

load_dotenv()

@dataclass
class LLMConfig:
    """LLM Provider Configuration"""
    PROVIDERS = {
        'openrouter': {
            'base_url': 'https://openrouter.io/api/v1',
            'env_var': 'OPENROUTER_API_KEY',
            'default_model': 'openai/gpt-4o-mini'
        },
        'openai': {
            'base_url': 'https://api.openai.com/v1',
            'env_var': 'OPENAI_API_KEY',
            'default_model': 'gpt-4o-mini'
        },
        'gemini': {
            'base_url': 'https://api.anthropic.com/v1',
            'env_var': 'ANTHROPIC_API_KEY',
            'default_model': 'claude-3-haiku'
        }
    }
    
    provider: str = os.getenv('LLM_PROVIDER', 'openrouter')
    model: str = None
    timeout: int = int(os.getenv('LLM_TIMEOUT', 30))
    max_retries: int = 3

    def __post_init__(self):
        if self.provider not in self.PROVIDERS:
            raise ValueError(f"Provider {self.provider} not supported")
        
        self.config = self.PROVIDERS[self.provider]
        self.api_key = os.getenv(self.config['env_var'])
        
        if not self.api_key:
            raise ValueError(f"Missing API key for {self.provider}")
        
        self.model = self.model or self.config['default_model']

# Global config instance
config = LLMConfig()
```

---

### Phase 2: Create Enhanced LLM Module

Create `llm_provider.py`:

```python
"""
Universal LLM provider module supporting OpenRouter, OpenAI, and more
"""

import requests
import logging
from typing import List, Dict, Any, Optional
from tenacity import retry, stop_after_attempt, wait_exponential
from config import config

logger = logging.getLogger(__name__)

class LLMProvider:
    """Unified interface for multiple LLM providers"""
    
    def __init__(self, config_override: Optional[Dict] = None):
        self.config = {**config.__dict__, **(config_override or {})}
        self.base_url = config.config['base_url']
        self.api_key = config.api_key
        self.headers = self._build_headers()
    
    def _build_headers(self) -> Dict[str, str]:
        """Build request headers based on provider"""
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        }
        
        if self.config['provider'] == 'openrouter':
            headers["HTTP-Referer"] = "https://github.com/alosh20hh-droid/visionpod"
            headers["X-Title"] = "VisionPod Smart Glasses AI"
        
        return headers
    
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=10)
    )
    def analyze_image(
        self,
        images: List[str],
        query: str,
        system_prompt: Optional[str] = None,
        temperature: float = 0.7,
        max_tokens: int = 1000
    ) -> str:
        """
        Analyze images with AI and answer query
        
        Args:
            images: List of base64 encoded images
            query: User query/question
            system_prompt: Custom system prompt
            temperature: Model temperature (0-1)
            max_tokens: Maximum response tokens
        
        Returns:
            AI response text
        
        Raises:
            requests.exceptions.RequestException: API error
        """
        
        if not system_prompt:
            system_prompt = "You are a helpful AI assistant with excellent vision capabilities. Analyze the provided images and answer questions about them accurately and concisely."
        
        messages = [
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": self._build_vision_content(images, query)
            }
        ]
        
        payload = {
            "model": self.config['model'],
            "messages": messages,
            "temperature": temperature,
            "max_tokens": max_tokens
        }
        
        logger.info(f"Calling {self.config['provider']} with {len(images)} images")
        
        try:
            response = requests.post(
                f"{self.base_url}/chat/completions",
                json=payload,
                headers=self.headers,
                timeout=self.config['timeout']
            )
            response.raise_for_status()
            
            result = response.json()
            
            if 'error' in result:
                raise ValueError(f"API Error: {result['error'].get('message', 'Unknown error')}")
            
            return result['choices'][0]['message']['content']
        
        except requests.exceptions.Timeout:
            logger.error(f"Request timeout after {self.config['timeout']}s")
            raise
        except requests.exceptions.RequestException as e:
            logger.error(f"Request failed: {e}")
            raise
    
    def _build_vision_content(self, images: List[str], query: str) -> List[Dict]:
        """Build content list with images and text"""
        content = []
        
        # Add text query
        content.append({
            "type": "text",
            "text": query
        })
        
        # Add images
        for base64_image in images:
            content.append({
                "type": "image",
                "image": {
                    "url": f"data:image/jpeg;base64,{base64_image}"
                }
            })
        
        return content
    
    def change_model(self, model: str):
        """Change the model at runtime"""
        self.config['model'] = model
        logger.info(f"Switched to model: {model}")
    
    def change_provider(self, provider: str):
        """Change the provider at runtime"""
        if provider not in config.PROVIDERS:
            raise ValueError(f"Provider {provider} not supported")
        
        self.config['provider'] = provider
        self.base_url = config.PROVIDERS[provider]['base_url']
        self.api_key = os.getenv(config.PROVIDERS[provider]['env_var'])
        self.headers = self._build_headers()
        
        logger.info(f"Switched to provider: {provider}")
```

---

### Phase 3: Update visionpod's app.py

Replace/enhance your current `app.py`:

```python
"""
Enhanced VisionPod with OpenRouter/Multi-provider LLM support
"""

from fastapi import FastAPI, File, Form, HTTPException, UploadFile
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.middleware.cors import CORSMiddleware
import os
from PIL import Image
import io
from dotenv import load_dotenv
import logging
from llm_provider import LLMProvider
import base64

# Setup
load_dotenv()
logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

app = FastAPI(
    title="VisionPod Smart Glasses",
    description="AI-powered smart glasses with real-time vision analysis"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize LLM provider
try:
    llm = LLMProvider()
    logger.info(f"LLM Provider initialized: {os.getenv('LLM_PROVIDER', 'openrouter')}")
except Exception as e:
    logger.error(f"Failed to initialize LLM provider: {e}")
    llm = None

# HTML UI (Arabic)
HTML = r"""<!doctype html>
<html lang="ar" dir="rtl">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>VisionPod — اختبار كاميرا الهاتف</title>
<style>
body { font-family: system-ui, sans-serif; max-width: 760px; margin: 0 auto; padding: 16px; background: #f6f7f8; }
main { background: white; padding: 18px; border-radius: 16px; box-shadow: 0 2px 14px #0001; }
h1 { font-size: 1.5rem; margin-top: 0; }
video, canvas { width: 100%; max-height: 55vh; background: #111; border-radius: 12px; }
button { border: 0; border-radius: 10px; padding: 12px 16px; font-size: 1rem; background: #185adb; color: white; }
button.secondary { background: #4b5563; }
button:disabled { opacity: .5; }
textarea { width: 100%; box-sizing: border-box; min-height: 74px; padding: 10px; border-radius: 10px; border: 1px solid #ccd1d8; }
.provider-select { padding: 8px; border-radius: 8px; margin: 10px 0; }
#status, #answer { white-space: pre-wrap; background: #f0f3f7; padding: 12px; border-radius: 10px; margin-top: 12px; }
.note { font-size: .92rem; color: #4b5563; }
</style>
</head>
<body>
<main>
<h1>VisionPod 📷 مع OpenRouter</h1>
<p>اختبر كاميرتك مع أحدث نماذج AI من OpenRouter</p>

<label><b>اختر مزود الخدمة:</b></label>
<select id="provider" class="provider-select">
<option value="openrouter">OpenRouter (موصى به)</option>
<option value="openai">OpenAI (مباشر)</option>
</select>

<label><b>اختر النموذج:</b></label>
<select id="model" class="provider-select">
<option value="openai/gpt-4o-mini">GPT-4o Mini (سريع وسعر منخفض)</option>
<option value="anthropic/claude-3-haiku">Claude 3 Haiku (رخيص جداً)</option>
<option value="openai/gpt-4">GPT-4 (الأكثر قدرة)</option>
</select>

<video id="video" autoplay playsinline></video>
<canvas id="canvas"></canvas>

<div>
<button id="start">تشغيل الكاميرا</button>
<button id="capture" disabled>التقاط صورة</button>
</div>

<label><b>سؤالك:</b></label>
<textarea id="question">صف لي ما تراه في الصورة</textarea>

<button id="analyze" disabled>تحليل الصورة</button>

<div id="status">الحالة: بانتظار تشغيل الكاميرا</div>

<h3>النتيجة</h3>
<div id="answer">لم يتم التحليل بعد</div>
</main>

<script>
const video = document.getElementById('video');
const canvas = document.getElementById('canvas');
const startBtn = document.getElementById('start');
const captureBtn = document.getElementById('capture');
const analyzeBtn = document.getElementById('analyze');
const statusBox = document.getElementById('status');
const answerBox = document.getElementById('answer');
const providerSelect = document.getElementById('provider');
const modelSelect = document.getElementById('model');

let stream = null;
let photoBlob = null;

startBtn.onclick = async () => {
try {
stream = await navigator.mediaDevices.getUserMedia({
video: { facingMode: { ideal: "environment" } }
});
video.srcObject = stream;
await video.play();
captureBtn.disabled = false;
statusBox.textContent = "الحالة: الكاميرا تعمل";
} catch (e) {
statusBox.textContent = "خطأ: " + e.message;
}
};

captureBtn.onclick = () => {
if (!video.videoWidth) return;
canvas.width = video.videoWidth;
canvas.height = video.videoHeight;
canvas.getContext('2d').drawImage(video, 0, 0);
canvas.toBlob(blob => {
photoBlob = blob;
analyzeBtn.disabled = false;
statusBox.textContent = "تم التقاط الصورة محليًا";
}, "image/jpeg", 0.82);
};

analyzeBtn.onclick = async () => {
if (!photoBlob) return;
analyzeBtn.disabled = true;
statusBox.textContent = "جارٍ التحليل...";

const data = new FormData();
data.append("image", photoBlob, "image.jpg");
data.append("question", document.getElementById('question').value);
data.append("provider", providerSelect.value);
data.append("model", modelSelect.value);

try {
const res = await fetch("/analyze", { method: "POST", body: data });
const result = await res.json();
if (!res.ok) throw new Error(result.detail);
answerBox.textContent = result.answer;
statusBox.textContent = "✅ نجح التحليل";
} catch (e) {
statusBox.textContent = "❌ خطأ: " + e.message;
} finally {
analyzeBtn.disabled = false;
}
};
</script>
</body>
</html>"""

# API Routes

@app.get("/", response_class=HTMLResponse)
def home():
    """Serve web UI"""
    return HTML

@app.get("/health")
def health():
    """Health check endpoint"""
    return {
        "status": "ok",
        "service": "VisionPod",
        "llm_provider": os.getenv("LLM_PROVIDER", "openrouter"),
        "llm_initialized": llm is not None
    }

@app.post("/analyze")
async def analyze(
    image: UploadFile = File(...),
    question: str = Form("صف هذه الصورة"),
    provider: str = Form(None),
    model: str = Form(None)
):
    """
    Analyze image with AI
    
    Query params:
    - image: uploaded image file
    - question: question about the image
    - provider: optional LLM provider override
    - model: optional model override
    """
    
    if not llm:
        raise HTTPException(status_code=503, detail="LLM provider not initialized")
    
    # Validate image
    if image.content_type not in ("image/jpeg", "image/png", "image/webp"):
        raise HTTPException(status_code=415, detail="نوع الصورة غير مدعوم")
    
    raw = await image.read()
    if len(raw) > 8 * 1024 * 1024:
        raise HTTPException(status_code=413, detail="الصورة كبيرة جداً")
    
    try:
        # Validate image
        img = Image.open(io.BytesIO(raw)).convert("RGB")
        
        # Encode to base64
        img_base64 = base64.b64encode(raw).decode('utf-8')
        
        # Override provider/model if requested
        if provider:
            llm.change_provider(provider)
        if model:
            llm.change_model(model)
        
        # Analyze
        response = llm.analyze_image(
            images=[img_base64],
            query=question,
            temperature=0.7,
            max_tokens=1000
        )
        
        return {
            "answer": response,
            "provider": llm.config['provider'],
            "model": llm.config['model'],
            "image_size": f"{img.width}x{img.height}"
        }
    
    except Exception as exc:
        logger.error(f"Analysis error: {exc}")
        raise HTTPException(
            status_code=502,
            detail=f"تعذر تحليل الصورة: {str(exc)}"
        )

if __name__ == "__main__":
    import uvicorn
    port = int(os.getenv("PORT", 8000))
    uvicorn.run(app, host="0.0.0.0", port=port, log_level="info")
```

---

### Phase 4: Update .env File

Create `.env` in your visionpod root:

```env
# OpenRouter Configuration
OPENROUTER_API_KEY=your_api_key_here
LLM_PROVIDER=openrouter

# Optional: OpenAI direct
# OPENAI_API_KEY=sk-...

# Application
LLM_TIMEOUT=30
PORT=8000
LOG_LEVEL=INFO
```

---

### Phase 5: Installation & Testing

```bash
# Install dependencies
pip install -r requirements.txt

# Test import
python -c "from llm_provider import LLMProvider; print('✅ Import successful')"

# Run application
python app.py

# Visit http://localhost:8000
```

---

## 🎨 Advanced Features

### 1. Dynamic Provider Switching

```python
# During runtime
@app.post("/switch-provider")
def switch_provider(provider: str, model: str = None):
    try:
        llm.change_provider(provider)
        if model:
            llm.change_model(model)
        return {"status": "ok", "provider": provider, "model": model}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
```

### 2. Fallback Mechanism

```python
def analyze_with_fallback(images, query):
    providers = ['openrouter', 'openai']
    
    for provider in providers:
        try:
            llm.change_provider(provider)
            return llm.analyze_image(images, query)
        except Exception as e:
            logger.warning(f"Failed with {provider}, trying next...")
            continue
    
    raise Exception("All providers failed")
```

### 3. Cost Tracking

```python
class CostTracker:
    COSTS = {
        'openai/gpt-4o-mini': 0.00015,  # per token
        'anthropic/claude-3-haiku': 0.00008,
    }
    
    def estimate_cost(self, model: str, input_tokens: int, output_tokens: int):
        cost_per_token = self.COSTS.get(model, 0.0001)
        return (input_tokens + output_tokens) * cost_per_token
```

---

## 📈 Performance Tips

1. **Cache base64 encoding** - Encode once, use multiple times
2. **Batch requests** - Send multiple images together
3. **Use cheaper models** - For simple tasks use gpt-4o-mini
4. **Implement timeout** - Prevent hanging requests
5. **Log everything** - Track costs and performance

---

## ✅ Checklist

- [ ] Get OpenRouter API key
- [ ] Create .env file with API key
- [ ] Install new dependencies
- [ ] Create config.py
- [ ] Create llm_provider.py
- [ ] Update app.py
- [ ] Test locally
- [ ] Deploy to production
- [ ] Monitor costs
- [ ] Optimize models

---

**Status:** Ready for visionpod integration ✅
