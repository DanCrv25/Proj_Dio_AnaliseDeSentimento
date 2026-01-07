# Proj_Dio_AnaliseDeSentimento
Análise de sentimento de texto 

🎙️ Análise de Sentimento a partir de Áudio com Azure AI Speech

Este projeto demonstra, de forma simples e educativa, como:

Converter áudio em texto usando Azure AI Speech (Speech to Text)

Analisar o sentimento do texto usando Azure AI Language (Sentiment Analysis)

O objetivo é aprendizado, não produção em larga escala.

🎯 Objetivo

Transformar um áudio curto (ex.: gravação de um cliente) em:

texto transcrito

classificação de sentimento (positive, neutral, negative)

scores de confiança

🧱 Arquitetura (simples)
Arquivo WAV
   ↓
Azure AI Speech (Speech to Text)
   ↓
Texto transcrito
   ↓
Azure AI Language (Sentiment Analysis)
   ↓
Sentimento + scores


Tudo acontece localmente via Python, chamando serviços do Azure.

📦 Tecnologias usadas

Azure AI Speech

Conversão de fala → texto

Azure AI Language

Análise de sentimento

Python 3.9+

SDKs oficiais da Microsoft

☁️ Recursos necessários no Azure

criar dois recursos no Azure Portal:

1️⃣ Azure AI Speech

Tipo: Speech

Usado para: transcrição de áudio

SPEECH_KEY

SPEECH_REGION

2️⃣ Azure AI Language

Tipo: Language (ou AI Services)

Usado para: análise de sentimento

LANGUAGE_KEY

LANGUAGE_ENDPOINT


🗂️ Estrutura do repositório
.
├── README.md
├── sentiment_from_audio.py
├── sample.wav
└── requirements.txt

🛠️ Instalação
1) Criar ambiente virtual (opcional, recomendado)
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows

2) Instalar dependências
pip install azure-cognitiveservices-speech azure-ai-textanalytics

usando requirements.txt:

azure-cognitiveservices-speech
azure-ai-textanalytics

pip install -r requirements.txt

🔐 Configuração das variáveis de ambiente

As chaves do Azure como variáveis de ambiente.

Linux / Mac
export SPEECH_KEY="sua_chave_speech"
export SPEECH_REGION="sua_regiao"
export LANGUAGE_KEY="sua_chave_language"
export LANGUAGE_ENDPOINT="https://seu-endpoint.cognitiveservices.azure.com/"

Windows (PowerShell)
setx SPEECH_KEY "sua_chave_speech"
setx SPEECH_REGION "sua_regiao"
setx LANGUAGE_KEY "sua_chave_language"
setx LANGUAGE_ENDPOINT "https://seu-endpoint.cognitiveservices.azure.com/"

▶️ Como executar

Um arquivo WAV curto na raiz do projeto

Exemplo: sample.wav

Idioma recomendado: Português

Áudio limpo, sem muito ruído

Execute o script:

python sentiment_from_audio.py

🧠 O que o script faz (passo a passo)
1️⃣ Speech to Text

Envia o áudio para o Azure AI Speech

Recebe a transcrição do áudio

2️⃣ Sentiment Analysis

Envia o texto transcrito para o Azure AI Language

Recebe:

sentimento (positive, neutral, negative)

scores de confiança

🧪 Exemplo de saída
TRANSCRIÇÃO:
Estou muito insatisfeito, meu pedido ainda não chegou.

SENTIMENTO:
negative

SCORES:
positive=0.02, neutral=0.11, negative=0.87

📄 Código principal (sentiment_from_audio.py)
import os
import azure.cognitiveservices.speech as speechsdk
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

# ================= CONFIG =================
SPEECH_KEY = os.environ["SPEECH_KEY"]
SPEECH_REGION = os.environ["SPEECH_REGION"]

LANGUAGE_KEY = os.environ["LANGUAGE_KEY"]
LANGUAGE_ENDPOINT = os.environ["LANGUAGE_ENDPOINT"]

WAV_FILE = "sample.wav"

# ============ SPEECH TO TEXT ==============
speech_config = speechsdk.SpeechConfig(
    subscription=SPEECH_KEY,
    region=SPEECH_REGION,
    speech_recognition_language="pt-BR"
)

audio_config = speechsdk.audio.AudioConfig(filename=WAV_FILE)
recognizer = speechsdk.SpeechRecognizer(
    speech_config=speech_config,
    audio_config=audio_config
)

result = recognizer.recognize_once()

if result.reason != speechsdk.ResultReason.RecognizedSpeech:
    raise RuntimeError("Falha ao reconhecer o áudio")

transcript = result.text
print("TRANSCRIÇÃO:")
print(transcript)

# ============ SENTIMENT ANALYSIS ==========
client = TextAnalyticsClient(
    endpoint=LANGUAGE_ENDPOINT,
    credential=AzureKeyCredential(LANGUAGE_KEY)
)

response = client.analyze_sentiment(
    documents=[transcript],
    language="pt"
)[0]

print("\nSENTIMENTO:")
print(response.sentiment)

print("\nSCORES:")
print(response.confidence_scores)
