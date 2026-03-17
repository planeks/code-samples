# LLM Integration (OpenAI + Django)

A production example of integrating OpenAI into a Django telephony application for AI-powered call analysis. When a phone call ends, a Celery task downloads the recording and sends it to OpenAI's transcription API with speaker diarization via a gateway class. The transcribed text is then passed to a second OpenAI call that uses structured outputs -- a Pydantic schema is provided as the `response_format`, so the LLM returns a validated Python object containing the call summary, key points, action items, and sentiment analysis. The gateway layer is built on an abstract interface with generics, making it possible to swap AI providers without changing business logic. The entire pipeline runs asynchronously through chained Celery tasks, with results persisted to Django models.

## AI Provider Interface

An abstract base class parameterized with generics on both the client type (e.g. `OpenAI`) and the response type (any Pydantic model). This makes the integration provider-agnostic -- a new provider can be added by implementing this interface without touching the service layer.

_services/ai/interfaces.py_

```python
from abc import ABC, abstractmethod
from typing import Any, Generic, TypeVar

from pydantic import BaseModel

from services.ai.types import AIInitParams, AIRequestParams, Providers

ClientType = TypeVar("ClientType")
ResponseType = TypeVar("ResponseType", bound=BaseModel)


class AIProviderInterface(ABC, Generic[ClientType, ResponseType]):
    _client: ClientType
    _provider: Providers

    def __init__(self, init_params: AIInitParams) -> None:
        super().__init__()

    @abstractmethod
    def _make_request(self, request_params: AIRequestParams) -> Any: ...

    @abstractmethod
    def run(self, request_params: AIRequestParams) -> ResponseType: ...

    @abstractmethod
    def _adapt_response_to_domain(self, response: Any) -> ResponseType: ...
```

## Type Definitions

Pydantic models that separate initialization parameters (API key, model name) from request parameters (system instructions, user prompt). The `Providers` enum allows tracking which provider was used for a given request.

_services/ai/types.py_

```python
from django.db import models
from django.utils.translation import gettext_lazy as _

from pydantic import BaseModel


class AIInitParams(BaseModel):
    api_key: str
    ai_model: str


class AIRequestParams(BaseModel):
    instructions: str
    prompt: str


class Providers(models.IntegerChoices):
    ANY_PROVIDER = 0, _("Any Provider")
    OPEN_AI = 1, _("Open AI")
```

## OpenAI Gateway

The concrete implementation of the AI provider interface. The key method is `_make_request`, which uses `client.beta.chat.completions.parse()` with a Pydantic model as `response_format` -- this constrains the LLM to return JSON matching the schema, which the SDK automatically parses into a typed Python object. The same class also handles audio transcription with configurable model and chunking strategy.

_services/ai/gateways.py_

```python
import logging
from typing import Any, BinaryIO, Optional, Type, TypeVar

from openai import OpenAI
from pydantic import BaseModel

from services.ai.interfaces import AIProviderInterface
from services.ai.types import AIInitParams, AIRequestParams, Providers

logger = logging.getLogger(__name__)

ResponseType = TypeVar("ResponseType", bound=BaseModel)


class OpenAIGateway(AIProviderInterface[OpenAI, ResponseType]):
    _ai_model: str
    _client: OpenAI
    _provider = Providers.OPEN_AI
    _response_format: Type[ResponseType]

    def __init__(
        self, init_params: AIInitParams, response_format: Type[ResponseType]
    ) -> None:
        self._ai_model = init_params.ai_model
        self._client = OpenAI(api_key=init_params.api_key)
        self._response_format = response_format

    def _make_request(self, request_params: AIRequestParams) -> ResponseType:
        logger.debug("Sending OpenAI request")
        completion = self._client.beta.chat.completions.parse(
            model=self._ai_model,
            messages=[
                {"role": "system", "content": request_params.instructions},
                {"role": "user", "content": request_params.prompt},
            ],
            response_format=self._response_format,
        )
        return completion.choices[0].message.parsed

    def _adapt_response_to_domain(
        self, response: ResponseType
    ) -> ResponseType:
        return response

    def run(self, request_params: AIRequestParams) -> ResponseType:
        response = self._make_request(request_params)
        return self._adapt_response_to_domain(response)

    def transcribe_audio(
        self,
        audio_file: BinaryIO,
        model: str = "whisper-1",
        response_format: str = "json",
        chunking_strategy: Optional[str] = None,
    ) -> Any:
        params = {
            "model": model,
            "file": audio_file,
            "response_format": response_format,
        }
        if chunking_strategy:
            params["chunking_strategy"] = chunking_strategy

        transcript_response = self._client.audio.transcriptions.create(
            **params
        )
        logger.debug("Audio transcription completed")
        return transcript_response
```

## Structured Output Schema

The Pydantic model passed as `response_format` to the OpenAI SDK. This constrains the LLM to return JSON matching this schema exactly, which is then automatically parsed into a typed Python object -- no manual JSON parsing needed. The `Field` descriptions serve double duty: they document the code and guide the LLM's output.

_communications/types.py_

```python
from pydantic import BaseModel, Field


class UnifiedAnalysisResponse(BaseModel):
    summary: str = Field(
        ...,
        description="Brief summary of the call conversation",
    )
    key_points: list[str] = Field(
        default_factory=list,
        description="Key discussion points from the call",
    )
    action_items: list[str] = Field(
        default_factory=list,
        description="Action items or next steps identified",
    )
    sentiment: str = Field(
        description=(
            "Overall sentiment: very_positive, positive, neutral, "
            "negative, very_negative, or mixed"
        )
    )
    sentiment_score: float = Field(
        description=(
            "Sentiment score from -1.0 (very negative) to 1.0 (very positive)"
        )
    )
    sentiment_confidence: float = Field(
        description="Confidence in sentiment analysis from 0.0 to 1.0"
    )
```

## Prompt Configuration

System prompts, model settings, and speaker mapping are defined as module-level constants. The unified analysis prompt combines what were originally separate API calls (summary + sentiment) into one, reducing latency and cost. Speaker diarization labels are mapped to business-meaningful names.

_communications/settings.py_

```python
# OpenAI Transcription Settings
OPENAI_TRANSCRIPTION_MODEL = "gpt-4o-transcribe-diarize"
OPENAI_TRANSCRIPTION_RESPONSE_FORMAT = "diarized_json"
OPENAI_TRANSCRIPTION_CHUNKING_STRATEGY = "auto"

MAX_AUDIO_FILE_SIZE_BYTES = 25 * 1024 * 1024  # 25MB limit for OpenAI API

SPEAKER_MAPPING = {
    "A": "Agent",
    "B": "Customer",
    "C": "User",
    "D": "User",
}

UNIFIED_ANALYSIS_PROMPT = (
    "You are an assistant that comprehensively analyzes phone call transcripts. "
    "Perform ALL of the following analyses in a single response:\n\n"
    "1. SUMMARY: Provide a brief 2-3 sentence overview of the conversation, "
    "list 3-5 main discussion topics or important details (key_points), "
    "and identify specific tasks, follow-ups, or next steps mentioned (action_items).\n\n"
    "2. SENTIMENT: Analyze the overall sentiment of the conversation. "
    "Choose from positive, neutral or negative. "
    "Provide a numerical score from -1.0 (negative) to 1.0 (positive) "
    "and your confidence level from 0.0 to 1.0."
)
```

## Service Orchestration

The `TranscribeCallService` ties everything together. On initialization it creates an `OpenAIGateway` parameterized with `UnifiedAnalysisResponse` as the structured output type. The `execute` method runs the full pipeline: download the recording, transcribe it with diarization, format the transcript with speaker labels, and run AI analysis. The `_perform_analysis` method shows how the structured output is consumed -- the response is a typed Pydantic object with direct attribute access, mapped to Django model enums and persisted via `update_or_create`.

_communications/services.py_

```python
import logging
import os

from django.conf import settings
from django.utils import timezone

from analytics.models import SentimentAnalysis, Transcription
from communications.models import Call
from communications.settings import (
    OPENAI_TRANSCRIPTION_CHUNKING_STRATEGY,
    OPENAI_TRANSCRIPTION_MODEL,
    OPENAI_TRANSCRIPTION_RESPONSE_FORMAT,
    SPEAKER_MAPPING,
    UNIFIED_ANALYSIS_PROMPT,
)
from communications.types import UnifiedAnalysisResponse
from services.ai import AIInitParams, AIRequestParams, OpenAIGateway

logger = logging.getLogger(__name__)


class TranscribeCallService:
    def __init__(self, call: Call):
        self.call = call
        self.openai_gateway = OpenAIGateway(
            init_params=AIInitParams(
                api_key=settings.OPENAI_API_KEY,
                ai_model=settings.OPENAI_MODEL,
            ),
            response_format=UnifiedAnalysisResponse,
        )

    def execute(self) -> None:
        start_time = timezone.now()
        transcription = self._prepare_transcription()
        if not transcription:
            return

        temp_file_path = self._download_recording()
        if not temp_file_path:
            transcription.status = Transcription.TranscriptionStatus.FAILED
            transcription.save(update_fields=["status"])
            return

        try:
            # Step 1: Transcribe with OpenAI
            transcription_text = self._transcribe_audio(temp_file_path)

            # Step 2: Update transcription record
            processing_duration = (timezone.now() - start_time).total_seconds()
            transcription.text = transcription_text
            transcription.status = Transcription.TranscriptionStatus.COMPLETED
            transcription.processed_at = timezone.now()
            transcription.processing_duration_seconds = int(processing_duration)
            transcription.save()

            logger.info(
                f"Transcription completed for call {self.call.id} "
                f"in {processing_duration:.2f} seconds"
            )

            # Step 3: Perform AI analysis (summary, sentiment, keywords)
            self._perform_analysis(transcription_text)
        finally:
            if os.path.exists(temp_file_path):
                os.unlink(temp_file_path)

    def _transcribe_audio(self, audio_file_path: str) -> str:
        with open(audio_file_path, "rb") as audio_file:
            transcript_response = self.openai_gateway.transcribe_audio(
                audio_file=audio_file,
                model=OPENAI_TRANSCRIPTION_MODEL,
                response_format=OPENAI_TRANSCRIPTION_RESPONSE_FORMAT,
                chunking_strategy=OPENAI_TRANSCRIPTION_CHUNKING_STRATEGY,
            )
        return self._format_diarized_transcript(transcript_response)

    def _format_diarized_transcript(self, transcript_response) -> str:
        if hasattr(transcript_response, "segments") and transcript_response.segments:
            formatted_utterances = []
            current_speaker = None
            current_text_parts = []

            for segment in transcript_response.segments:
                speaker = getattr(segment, "speaker", None)
                text = getattr(segment, "text", "").strip()
                if not text:
                    continue

                # Map speaker IDs to Agent/Customer labels
                speaker_label = SPEAKER_MAPPING.get(
                    speaker,
                    speaker.replace("speaker_", "Speaker ") if speaker else None,
                )

                # If speaker changed, save the previous utterance
                if current_speaker is not None and current_speaker != speaker_label:
                    if current_text_parts:
                        combined_text = " ".join(current_text_parts)
                        if current_speaker:
                            formatted_utterances.append(
                                f"{current_speaker}: {combined_text}"
                            )
                        else:
                            formatted_utterances.append(combined_text)
                    current_text_parts = []

                current_speaker = speaker_label
                current_text_parts.append(text)

            # Don't forget the last utterance
            if current_text_parts and current_speaker is not None:
                combined_text = " ".join(current_text_parts)
                if current_speaker:
                    formatted_utterances.append(f"{current_speaker}: {combined_text}")
                else:
                    formatted_utterances.append(combined_text)

            return "\n".join(formatted_utterances)

        # Fallback to plain text if segments not available
        return getattr(transcript_response, "text", "")

    def _perform_analysis(self, transcription_text: str) -> None:
        if not settings.OPENAI_API_KEY:
            logger.warning("Skipping AI analysis: No OpenAI API key configured")
            return

        logger.info(f"Starting unified AI analysis for call {self.call.id}")

        prompt = f"Analyze this phone call transcription:\n\n{transcription_text}"

        # Single unified API call -- returns a typed Pydantic object
        analysis_response = self.openai_gateway.run(
            AIRequestParams(
                instructions=UNIFIED_ANALYSIS_PROMPT,
                prompt=prompt,
            )
        )

        # Format the summary from structured response fields
        summary_parts = [f"Summary: {analysis_response.summary}"]
        if analysis_response.key_points:
            summary_parts.append(
                "Key Points:\n- " + "\n- ".join(analysis_response.key_points)
            )
        if analysis_response.action_items:
            summary_parts.append(
                "Action Items:\n- " + "\n- ".join(analysis_response.action_items)
            )
        ai_summary = "\n\n".join(summary_parts)

        # Map LLM sentiment string to Django model enum
        sentiment_mapping = {
            "positive": SentimentAnalysis.Sentiment.POSITIVE,
            "negative": SentimentAnalysis.Sentiment.NEGATIVE,
            "neutral": SentimentAnalysis.Sentiment.NEUTRAL,
            "very_positive": SentimentAnalysis.Sentiment.VERY_POSITIVE,
            "very_negative": SentimentAnalysis.Sentiment.VERY_NEGATIVE,
            "mixed": SentimentAnalysis.Sentiment.MIXED,
        }
        sentiment = sentiment_mapping.get(
            analysis_response.sentiment.lower(),
            SentimentAnalysis.Sentiment.NEUTRAL,
        )

        # Persist analysis results
        SentimentAnalysis.objects.update_or_create(
            call=self.call,
            defaults={
                "sentiment": sentiment,
                "sentiment_score": analysis_response.sentiment_score,
                "confidence": analysis_response.sentiment_confidence,
                "ai_summary": ai_summary,
            },
        )

        logger.info(
            f"Completed unified AI analysis for call {self.call.id}: "
            f"sentiment={sentiment}, score={analysis_response.sentiment_score}"
        )
```

## Async Task Pipeline

Celery tasks chain the pipeline: after a call recording is uploaded to S3, the `transcribe_call` task is triggered automatically. Each task is independently retryable and idempotent -- the S3 upload checks if a key already exists, and the transcription service checks for an existing completed transcription before re-processing.

_communications/tasks.py_

```python
import logging

from celery import shared_task

from communications.models import Call
from communications.services import TranscribeCallService
from core.gateways import S3Gateway

logger = logging.getLogger(__name__)


@shared_task()
def upload_recording_to_s3(call_id: int):
    call_obj = Call.objects.select_related("company").filter(id=call_id).first()
    if not call_obj or not call_obj.recording_url:
        logger.error(f"Call not found or has no recording_url: {call_id}")
        return

    if call_obj.recording_s3_key:
        logger.info(f"Call {call_id} already has S3 recording")
        return

    s3_gateway = S3Gateway()
    s3_key = s3_gateway.upload_recording(
        recording_url=call_obj.recording_url,
        company_id=call_obj.company_id,
        call_sid=call_obj.twilio_call_sid,
    )

    call_obj.recording_s3_key = s3_key
    call_obj.save(update_fields=["recording_s3_key"])
    logger.info(f"Uploaded recording for call {call_id} to S3: {s3_key}")

    # Chain: after S3 upload completes, trigger transcription
    transcribe_call.delay(call_id)


@shared_task()
def transcribe_call(call_id: int):
    call_obj = Call.objects.select_related("company").filter(id=call_id).first()
    if not call_obj:
        logger.error(f"Call not found: {call_id}")
        return

    service = TranscribeCallService(call=call_obj)
    service.execute()
```
