# Hidden Modes --- Teleport, Voice, Fast Mode, Effort, Ultraplan

Claude Code has several modes beyond the standard REPL. Understanding these patterns lets you build similar capabilities into your own harness.

## Why this chapter exists

Most developers interact with Claude Code through the basic prompt-response loop. Beneath the surface, however, there are specialized modes that change *how* the harness communicates with the model, *what* tools are available, and *where* a session can run. Each mode is a self-contained pattern you can lift into your own agent.

---

## 1. Teleport Mode (Cross-Machine Session Transfer)

Sessions can be transferred between machines. The flow:

1. Serialize conversation messages to JSON
2. Create a git bundle of the working directory state
3. Upload session data via API
4. On the target machine: download session, apply git bundle, resume conversation

!!! info "When to use"
    Teleport is useful for handing off long-running sessions from a laptop to a cloud VM, or sharing in-progress work with a teammate who needs full conversation context.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class TeleportPayload:
    session_id: str
    messages: list[dict]
    branch_name: str
    git_bundle_path: str | None


async def teleport_export(session, working_dir: str) -> TeleportPayload:
    """Export current session for transfer to another machine."""
    import subprocess

    # Create git bundle of current branch
    branch = subprocess.check_output(
        ["git", "rev-parse", "--abbrev-ref", "HEAD"],
        cwd=working_dir,
        text=True,
    ).strip()

    bundle_path = f"/tmp/teleport-{session.session_id}.bundle"
    subprocess.run(
        ["git", "bundle", "create", bundle_path, branch],
        cwd=working_dir,
        check=True,
    )

    return TeleportPayload(
        session_id=session.session_id,
        messages=session.messages,
        branch_name=branch,
        git_bundle_path=bundle_path,
    )


async def teleport_import(
    payload: TeleportPayload, working_dir: str
) -> list[dict]:
    """Import a teleported session on the target machine."""
    import subprocess

    if payload.git_bundle_path:
        subprocess.run(
            ["git", "bundle", "unbundle", payload.git_bundle_path],
            cwd=working_dir,
            check=True,
        )
        subprocess.run(
            ["git", "checkout", payload.branch_name],
            cwd=working_dir,
            check=True,
        )

    # Resume with a system message noting the transfer
    resume_message = {
        "role": "user",
        "content": (
            "This session was continued from another machine. "
            f"Working directory: {working_dir}"
        ),
    }
    return [*payload.messages, resume_message]
```

---

## 2. Fast Mode (Priority Processing)

Fast mode provides faster inference at 6x the cost (Opus 4.6: $30/$150 per Mtok vs $5/$25). The harness manages a cooldown state so rate-limit errors degrade gracefully.

```python
from dataclasses import dataclass, replace
import time


@dataclass(frozen=True)
class FastModeState:
    enabled: bool = False
    status: str = "active"  # "active" | "cooldown"
    cooldown_until: float = 0.0
    cooldown_reason: str = ""

    def is_available(self) -> bool:
        if not self.enabled:
            return False
        if self.status == "cooldown":
            if time.time() >= self.cooldown_until:
                return True  # cooldown expired
            return False
        return True

    def with_cooldown(
        self, duration_seconds: float, reason: str
    ) -> "FastModeState":
        return replace(
            self,
            status="cooldown",
            cooldown_until=time.time() + duration_seconds,
            cooldown_reason=reason,
        )

    def check_and_reset(self) -> "FastModeState":
        """Return a new state with cooldown cleared if it has expired."""
        if self.status == "cooldown" and time.time() >= self.cooldown_until:
            return replace(self, status="active", cooldown_reason="")
        return self
```

Use with either SDK:

```python
# Fast mode sends a beta header to get priority processing
model = "claude-opus-4-6"

def build_fast_headers(fast_mode: FastModeState) -> dict[str, str]:
    if fast_mode.is_available():
        # Anthropic: add beta header
        # Cost: $30/$150 per Mtok instead of $5/$25
        return {"anthropic-beta": "fast-mode-2025-04-01"}
    return {}
```

---

## 3. Effort Control (Thinking Budget)

Effort levels control how much "thinking" the model does before responding. Higher effort means deeper reasoning at the cost of latency and tokens.

```python
from enum import Enum


class EffortLevel(Enum):
    LOW = "low"        # Quick, minimal overhead
    MEDIUM = "medium"  # Balanced (default for most models)
    HIGH = "high"      # Comprehensive, thorough
    MAX = "max"        # Maximum reasoning (Opus 4.6 only)


def resolve_effort(
    model: str, user_preference: str | None
) -> EffortLevel | None:
    """Resolve effort level from user preference and model capabilities."""
    if user_preference is None:
        # Model-specific defaults
        if "opus-4-6" in model:
            return EffortLevel.MEDIUM
        return None  # No effort parameter sent

    level = EffortLevel(user_preference)

    # MAX is only supported by Opus 4.6
    if level == EffortLevel.MAX and "opus-4-6" not in model:
        return EffortLevel.HIGH  # downgrade gracefully

    return level


def apply_effort_to_params(
    params: dict, effort: EffortLevel | None
) -> dict:
    """Return new params dict with effort-related fields set."""
    if effort is None:
        return params

    budget_map = {
        EffortLevel.LOW: 1024,
        EffortLevel.MEDIUM: 8192,
        EffortLevel.HIGH: 16384,
        EffortLevel.MAX: 31999,
    }

    return {
        **params,
        "thinking": {
            "type": "enabled",
            "budget_tokens": budget_map[effort],
        },
    }
```

!!! warning "Budget tokens are not output tokens"
    The `budget_tokens` field controls *internal reasoning* tokens. The model may still produce a long final answer regardless of the thinking budget.

---

## 4. Ultraplan (Keyword-Triggered Planning)

When a user types a specific keyword (e.g., "ultraplan"), the harness intercepts the input and switches into a structured planning mode before sending anything to the model.

```python
import re

ULTRAPLAN_KEYWORDS = frozenset({"ultraplan", "ultrareview"})


def find_trigger_keyword(text: str) -> str | None:
    """Detect planning keywords while skipping quoted strings and paths."""
    # Strip content inside quotes
    stripped = re.sub(r'["\'].*?["\']', "", text)
    # Strip file paths
    stripped = re.sub(r"/\S+", "", stripped)

    words = stripped.lower().split()
    for word in words:
        if word in ULTRAPLAN_KEYWORDS:
            return word
    return None


def handle_ultraplan(keyword: str, user_input: str, session) -> str:
    """Trigger planning mode when keyword detected."""
    if keyword == "ultraplan":
        session.enter_plan_mode()
        return f"Entering structured planning mode for: {user_input}"
    if keyword == "ultrareview":
        session.enter_review_mode()
        return f"Starting deep code review for: {user_input}"
    raise ValueError(f"Unknown trigger keyword: {keyword}")
```

---

## 5. Plan Mode (Structured Planning Before Implementation)

Plan mode restricts the model to read-only operations until a plan is approved. This prevents the agent from making changes before the user has reviewed its approach.

```python
from dataclasses import dataclass, field, replace


@dataclass(frozen=True)
class PlanMode:
    active: bool = False
    plan_text: str = ""
    approved: bool = False

    def enter(self) -> "PlanMode":
        return replace(self, active=True, approved=False, plan_text="")

    def exit_with_plan(self, plan: str) -> "PlanMode":
        """Exit plan mode with a plan for user approval."""
        return replace(self, plan_text=plan)

    def approve(self) -> "PlanMode":
        return replace(self, approved=True, active=False)

    def get_restricted_tools(self, all_tools: list) -> list:
        """In plan mode, only allow read-only tools."""
        if not self.active:
            return all_tools
        return [t for t in all_tools if t.is_read_only()]
```

!!! tip "Combine with Ultraplan"
    The `ultraplan` keyword from section 4 calls `session.enter_plan_mode()`, which activates the `PlanMode` guard above. Together they form a two-step pipeline: keyword detection triggers plan mode, and plan mode restricts tool access until the user approves.

---

## 6. Voice Mode (Audio Input)

Voice mode captures audio from a microphone, detects silence to know when the user has stopped speaking, and transcribes the result into text that feeds the normal query loop.

```python
import asyncio
from dataclasses import dataclass


@dataclass(frozen=True)
class VoiceConfig:
    sample_rate: int = 16_000       # 16kHz
    silence_threshold_ms: int = 1500
    max_recording_seconds: int = 120


async def capture_voice_input(config: VoiceConfig) -> str:
    """Capture audio input and transcribe using STT."""
    try:
        audio_data = await _native_capture(config)
    except ImportError:
        audio_data = await _sox_capture(config)

    transcript = await _transcribe(audio_data)
    return transcript


async def _sox_capture(config: VoiceConfig) -> bytes:
    """Fallback audio capture using SoX rec command."""
    silence_secs = str(config.silence_threshold_ms / 1000)
    proc = await asyncio.create_subprocess_exec(
        "rec", "-q",
        "-r", str(config.sample_rate),
        "-c", "1", "-b", "16", "-t", "wav", "-",
        "silence", "1", "0.1", "1%",
        "1", silence_secs, "1%",
        stdout=asyncio.subprocess.PIPE,
    )
    audio_data, _ = await proc.communicate()
    return audio_data


async def _native_capture(config: VoiceConfig) -> bytes:
    """Platform-native audio capture (requires sounddevice)."""
    import sounddevice as sd  # type: ignore[import-untyped]
    import numpy as np

    frames: list[np.ndarray] = []
    max_frames = config.sample_rate * config.max_recording_seconds

    def callback(indata, _frame_count, _time_info, _status):
        frames.append(indata.copy())

    with sd.InputStream(
        samplerate=config.sample_rate,
        channels=1,
        dtype="int16",
        callback=callback,
    ):
        while sum(len(f) for f in frames) < max_frames:
            await asyncio.sleep(0.1)

    return np.concatenate(frames).tobytes()


async def _transcribe(audio_data: bytes) -> str:
    """Placeholder --- swap in your STT provider."""
    raise NotImplementedError("Plug in Whisper, Deepgram, or another STT API")
```

---

## Build it yourself

Pick the modes that match your product:

| Mode | Core idea | Key mechanism |
|------|-----------|---------------|
| **Teleport** | Serialize session + git state | `git bundle` + JSON export |
| **Fast mode** | Priority tier with higher pricing | Beta header + cooldown FSM |
| **Effort** | Map effort levels to model params | Thinking budget tokens |
| **Ultraplan** | Keyword detection | Intercept input, enter plan mode |
| **Plan mode** | Read-only until approved | Tool filtering by `is_read_only()` |
| **Voice** | Audio capture to text | SoX / sounddevice + STT |

Each mode is independent --- you can adopt one without the others, or compose them (e.g., voice input triggers ultraplan which activates plan mode with low effort).
