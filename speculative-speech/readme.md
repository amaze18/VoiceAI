# Graph Based User Journey Tracking for Voice AI

## Overview

This document explains a graph based customer journey tracking system for conversational and voice AI systems.

The system uses Bayesian probability updates to estimate:

- Current conversation stage
- Likely next stage
- Stability of the conversation
- Readiness for speculative execution

The design is inspired by NVIDIA speculative speech processing concepts.

Reference:

- NVIDIA Speculative Speech Processing  
  https://github.com/NVIDIA/voice-agent-examples/blob/main/docs/SPECULATIVE_SPEECH_PROCESSING.md

---

# Problem Statement

Traditional conversational systems often use fixed intent classification.

Example:

```text
User said X -> Intent Y
