# Agent Architecture — AI-Powered Adaptive Spoken English Learning Platform

This document defines the **complete agentic system design** for the platform, covering the number of agents required, their roles, the orchestration pattern, LangGraph state machines, inter-agent communication, and tool assignments. All diagrams are in **Mermaid** syntax compatible with Eraser.io.

---

## Summary: How Many Agents?

**Total: 11 Agents**

| # | Agent Name | Type | LangGraph Role | Primary Responsibility |
|---|-----------|------|---------------|----------------------|
| 1 | Assessment Orchestrator | **Orchestrator** | `StateGraph` root | Drives the full learner assessment pipeline end-to-end |
| 2 | Recruiter Screening Orchestrator | **Orchestrator** | `StateGraph` root | Drives the full recruiter candidate-screening pipeline |
| 3 | Question Selector Agent | Sub-Agent | Node in Assessment graph | Selects equivalent-form questions by level, pathway, and history |
| 4 | Speech Analysis Agent | Sub-Agent | Node in Assessment graph | Coordinates STT transcription + acoustic feature extraction |
| 5 | Rule-Based Evaluator Agent | Sub-Agent | Node in Assessment graph | Applies grammar, lexical, fluency rule-based scoring |
| 6 | AI Coherence Evaluator Agent | Sub-Agent | Node in Assessment graph | Makes the ≤2 Vertex AI calls for coherence/relevance scoring |
| 7 | Feedback Composer Agent | Sub-Agent | Node in Assessment graph | Synthesises all scores into corrected transcripts, explanations, model answers |
| 8 | Learning Pathway Agent | Sub-Agent | Node in Assessment graph | Generates personalised weekly learning roadmaps from weakness profiles |
| 9 | Risk Monitor Agent | **Background Agent** | Scheduled `StateGraph` | Continuously scans for stagnation/decline; fires teacher alerts |
| 10 | Classifier Agent | **Infrastructure Agent** | Pre-orchestrator gate | Classifies every inbound request by role, intent, and pathway; routes to the correct orchestrator |
| 11 | Context Manager Agent | **Infrastructure Agent** | Cross-cutting shared service | Manages short-term (Redis) and long-term (PostgreSQL) context windows for all orchestrators |

> **Design Pattern Used:** Hierarchical **Orchestrator → Sub-Agent** pattern implemented with **LangGraph `StateGraph`** for orchestrators and **LangChain `Tool`** definitions for sub-agents. Agents 10 and 11 act as **cross-cutting infrastructure** that every orchestrator uses before starting its own state machine. All traces flow through **LangSmith** for observability and cost tracking.

---

## 1. Agent Roles & Responsibilities

### Orchestrators

#### Agent 1 — Assessment Orchestrator
- **Trigger**: Learner initiates an assessment session via the frontend
- **Responsibility**: Manages the full state machine from profiling → question selection → speech capture → scoring → feedback → pathway generation
- **Owns**: `AssessmentState` (LangGraph state object)
- **Coordinates**: Agents 3, 4, 5, 6, 7, 8
- **Conditional branching**: Routes to CEFR pathway or IELTS pathway based on learner goal
- **Error handling**: Retries on STT failure; gracefully degrades AI scoring if Vertex AI is unavailable

#### Agent 2 — Recruiter Screening Orchestrator
- **Trigger**: Recruiter initiates a candidate screening session
- **Responsibility**: Manages the full recruiter flow from rubric loading → question selection → candidate recording → scoring → rubric mapping → ATS export
- **Owns**: `ScreeningState` (LangGraph state object)
- **Coordinates**: Agents 3, 4, 5, 6
- **Conditional branching**: Routes to job-specific rubric mapping based on screening profile

### Sub-Agents

#### Agent 3 — Question Selector Agent
- **Tools**: `query_question_bank`, `filter_by_level`, `filter_by_history`, `randomise_equivalent_form`
- **Input**: User CEFR level / IELTS band, pathway, previous question IDs
- **Output**: Ordered list of 3 equivalent-form questions (Part 1, 2, 3)
- **Rule**: Never repeats a question answered in the last 3 assessments

#### Agent 4 — Speech Analysis Agent
- **Tools**: `call_google_stt`, `extract_acoustic_features`, `normalise_transcript`
- **Input**: Encrypted audio file reference
- **Output**: Transcript text + acoustic metrics (speech rate, pause count, filler word count, articulation score)
- **Constraint**: Operates asynchronously; returns a job ID for polling

#### Agent 5 — Rule-Based Evaluator Agent
- **Tools**: `score_grammar`, `score_lexical_diversity`, `score_speech_rate`, `score_fillers`, `score_pause_patterns`, `map_to_cefr`, `map_to_ielts_band`
- **Input**: Transcript text + acoustic metrics from Agent 4
- **Output**: Structured scoring object `{grammar, lexical, fluency, pronunciation_proxy, composite_rule_score, mapped_level}`
- **Constraint**: 100% deterministic — no AI calls; fully auditable

#### Agent 6 — AI Coherence Evaluator Agent
- **Tools**: `call_vertex_ai_coherence`, `call_vertex_ai_relevance`, `log_api_cost`
- **Input**: Transcript text + question context
- **Output**: `{coherence_score, relevance_score, api_calls_used, cost_usd}`
- **Constraint**: Hard cap of **2 API calls per assessment**; fails gracefully by returning `null` if budget exceeded

#### Agent 7 — Feedback Composer Agent
- **Tools**: `generate_corrected_transcript`, `generate_grammar_explanation`, `generate_vocabulary_suggestions`, `generate_model_answer`, `build_metric_chart_data`
- **Input**: Original transcript + scores from Agents 5 and 6
- **Output**: Complete `FeedbackPayload` including corrected text, explanations, model answer, chart-ready metrics
- **Note**: Uses rule templates for 80% of feedback; LLM only for model answer generation (counts toward the 2-call budget)

#### Agent 8 — Learning Pathway Agent
- **Tools**: `analyse_weakness_profile`, `query_activity_bank`, `filter_by_history`, `build_weekly_roadmap`, `avoid_repetition`
- **Input**: Scoring history + weakness profile + learner goals + activity completion log
- **Output**: 7-day activity plan with activities tagged by skill, level, topic, and difficulty
- **Rule**: No repeated activity within the last 4-week window

#### Agent 9 — Risk Monitor Agent (Background)
- **Trigger**: Runs on a daily cron schedule (LangGraph scheduled execution)
- **Tools**: `fetch_recent_scores`, `compare_against_baseline`, `calculate_stagnation_index`, `flag_at_risk_student`, `send_teacher_alert`
- **Input**: All learner score records for the past 30 days
- **Output**: At-risk student flags written to database + teacher notifications fired
- **Threshold rule**: Flag if no score improvement over 3 consecutive assessments OR score decreases >10%

### Infrastructure / Supporting Agents

#### Agent 10 — Classifier Agent
- **Position in architecture**: Sits at the API Gateway boundary, **before** any orchestrator is invoked
- **Trigger**: Every inbound request from any user (Learner, Teacher, Recruiter, Admin, Research Analyst)
- **Responsibility**: Parses the request, identifies the user's role, intent, and session type, then routes to the correct orchestrator or service. Also pre-classifies the learner pathway (CEFR vs. IELTS) so the Assessment Orchestrator receives a fully labelled context object and does not need to re-ask
- **Tools**: `decode_jwt_claims`, `classify_user_role`, `classify_intent`, `classify_session_type`, `classify_learning_pathway`, `route_to_orchestrator`, `return_clarification_request`
- **Input**: Raw HTTP request headers + JWT token + request body
- **Output**: `ClassifiedRequest` object — `{user_id, role, intent, session_type, pathway, orchestrator_target}`
- **Failure mode**: If intent cannot be determined with confidence ≥ 0.85, returns a structured clarification prompt to the frontend rather than routing blindly

#### Agent 11 — Context Manager Agent
- **Position in architecture**: Cross-cutting shared service called by **every orchestrator** at the start and end of each state machine execution
- **Trigger**: Called by Assessment Orchestrator, Recruiter Screening Orchestrator, and Risk Monitor Agent at `START` and `END` of every run
- **Responsibility**: Provides two-tier memory management —
  - **Short-term (Redis)**: Stores in-flight session state so that an interrupted assessment (page refresh, network drop) can be resumed without re-processing completed steps
  - **Long-term (PostgreSQL)**: Maintains a structured `UserContextProfile` (full assessment history, weakness profile, pathway history, activity completion log) so orchestrators always have a rich, up-to-date context window without querying multiple tables
- **Tools**: `store_session_context`, `retrieve_session_context`, `clear_session`, `get_user_context_profile`, `update_weakness_profile`, `append_assessment_to_history`, `prune_stale_sessions`
- **Input (read)**: `user_id` + `session_id` → returns `UserContextProfile` + `SessionState`
- **Input (write)**: Any orchestrator's partial or final state object
- **Output**: Enriched context object OR confirmation of successful state write
- **Context window rule**: The `UserContextProfile` returned to LLM-backed agents (Agents 6 and 7) is trimmed to the last 3 assessments to keep token usage within the Vertex AI context window budget

---

## 2. Orchestrator–Sub-Agent Architecture Overview

```mermaid
graph TB
    subgraph Triggers["External Triggers"]
        L([👤 Learner])
        T([🎓 Teacher])
        R([🏢 Recruiter])
        CRON([⏱️ Daily Cron])
    end

    subgraph InfraAgents["Infrastructure Agents (Cross-Cutting)"]
        CLS["🔀 Agent 10\nClassifier Agent\nRoute & Intent Classification"]
        CTX["🗄️ Agent 11\nContext Manager Agent\nRedis + PostgreSQL Memory"]
    end

    subgraph Orchestrators["Orchestrators (LangGraph StateGraphs)"]
        ORC1["🧠 Agent 1\nAssessment Orchestrator\nLangGraph StateGraph"]
        ORC2["🧠 Agent 2\nRecruiter Screening Orchestrator\nLangGraph StateGraph"]
        ORC9["🔁 Agent 9\nRisk Monitor Agent\nScheduled LangGraph"]
    end

    subgraph SubAgents["Sub-Agents (LangChain Tools)"]
        SA3["🔍 Agent 3\nQuestion Selector Agent"]
        SA4["🎙️ Agent 4\nSpeech Analysis Agent"]
        SA5["📏 Agent 5\nRule-Based Evaluator Agent"]
        SA6["🤖 Agent 6\nAI Coherence Evaluator Agent\n≤2 Vertex AI calls"]
        SA7["✍️ Agent 7\nFeedback Composer Agent"]
        SA8["🗺️ Agent 8\nLearning Pathway Agent"]
    end

    subgraph ExternalServices["External Services"]
        VERTEXAI[Google Vertex AI\nLLM Endpoint]
        STT[Google Cloud STT]
        LANGSMITH[LangSmith\nTracing & Observability]
        DB[(PostgreSQL\nData Store)]
        REDIS[(Redis\nSession Cache)]
        ATS[ATS API\nGreenhouse / Lever]
    end

    %% All user requests first hit the Classifier Agent
    L -->|HTTP Request| CLS
    T -->|HTTP Request| CLS
    R -->|HTTP Request| CLS
    CRON -->|Daily Run| ORC9

    %% Classifier routes to correct orchestrator
    CLS -->|role=Learner intent=assessment| ORC1
    CLS -->|role=Recruiter intent=screening| ORC2

    %% Every orchestrator reads/writes context via Context Manager
    CTX -->|UserContextProfile + SessionState| ORC1
    CTX -->|UserContextProfile + SessionState| ORC2
    CTX -->|Learner history| ORC9
    ORC1 -->|Write session state| CTX
    ORC2 -->|Write session state| CTX

    %% Assessment Orchestrator delegates to sub-agents
    ORC1 -->|Select Questions| SA3
    ORC1 -->|Process Speech| SA4
    ORC1 -->|Rule-based Score| SA5
    ORC1 -->|AI Score| SA6
    ORC1 -->|Compose Feedback| SA7
    ORC1 -->|Generate Pathway| SA8

    %% Recruiter Orchestrator delegates to sub-agents
    ORC2 -->|Select Questions| SA3
    ORC2 -->|Process Speech| SA4
    ORC2 -->|Rule-based Score| SA5
    ORC2 -->|AI Score| SA6

    %% Risk Monitor
    ORC9 -->|Read Scores| DB
    ORC9 -->|Write Flags| DB

    %% Context Manager storage backends
    CTX -->|Short-term state| REDIS
    CTX -->|Long-term history| DB

    %% Sub-agent external calls
    SA4 -->|Transcribe Audio| STT
    SA6 -->|Coherence API| VERTEXAI
    SA7 -->|Model Answer API| VERTEXAI

    %% All agents traced
    ORC1 -.->|Trace| LANGSMITH
    ORC2 -.->|Trace| LANGSMITH
    ORC9 -.->|Trace| LANGSMITH
    CLS -.->|Trace| LANGSMITH
    CTX -.->|Trace| LANGSMITH

    %% Data persistence
    SA3 --> DB
    SA5 --> DB
    SA7 --> DB
    SA8 --> DB
    ORC2 -->|Push Results| ATS

    style Triggers fill:#dbeafe,stroke:#3b82f6
    style InfraAgents fill:#f3e8ff,stroke:#a855f7
    style Orchestrators fill:#fce7f3,stroke:#ec4899
    style SubAgents fill:#dcfce7,stroke:#22c55e
    style ExternalServices fill:#fff7ed,stroke:#f97316
```

---

## 3. LangGraph State Machine — Assessment Orchestrator (Agent 1)

> State machine governing the full learner assessment pipeline.

```mermaid
stateDiagram-v2
    [*] --> IDLE

    IDLE --> PROFILING : Learner starts session

    PROFILING --> PATHWAY_DECISION : Profile collected
    note right of PROFILING
        Collect: education level,
        confidence, goals, timeline
    end note

    PATHWAY_DECISION --> CEFR_TRACK : Goal = General / School
    PATHWAY_DECISION --> IELTS_TRACK : Goal = IELTS / Career / Interview

    CEFR_TRACK --> QUESTION_SELECTION : CEFR level set
    IELTS_TRACK --> QUESTION_SELECTION : IELTS band set

    QUESTION_SELECTION --> RECORDING : Questions served to frontend
    note right of QUESTION_SELECTION
        Agent 3 — Question Selector
        Picks 3 equivalent-form Qs
        avoiding history
    end note

    RECORDING --> SPEECH_PROCESSING : Audio submitted
    note right of RECORDING
        Frontend handles mic capture
        Audio encrypted → Cloud Storage
    end note

    SPEECH_PROCESSING --> RULE_SCORING : Transcript + features ready
    note right of SPEECH_PROCESSING
        Agent 4 — Speech Analysis
        STT + Acoustic Features
    end note

    RULE_SCORING --> AI_SCORING : Rule scores computed
    note right of RULE_SCORING
        Agent 5 — Rule-Based Evaluator
        Grammar, Lexical, Fluency
        100% deterministic
    end note

    AI_SCORING --> SCORE_AGGREGATION : AI scores computed (≤2 calls)
    note right of AI_SCORING
        Agent 6 — AI Coherence Evaluator
        Coherence + Relevance via Vertex AI
        Hard cap: 2 API calls
    end note

    SCORE_AGGREGATION --> FEEDBACK_GENERATION : Composite score finalised
    SCORE_AGGREGATION --> IMPROVEMENT_INDEX : If reassessment

    FEEDBACK_GENERATION --> PATHWAY_GENERATION : Feedback payload ready
    note right of FEEDBACK_GENERATION
        Agent 7 — Feedback Composer
        Corrected transcript, grammar notes,
        vocabulary, model answer, charts
    end note

    PATHWAY_GENERATION --> COMPLETE : Weekly roadmap ready
    note right of PATHWAY_GENERATION
        Agent 8 — Learning Pathway Agent
        Weakness analysis → Activity selection
    end note

    IMPROVEMENT_INDEX --> COMPLETE : Index computed

    COMPLETE --> [*]

    SPEECH_PROCESSING --> STT_RETRY : STT failure
    STT_RETRY --> SPEECH_PROCESSING : Retry (max 3)
    STT_RETRY --> GRACEFUL_FAIL : All retries exhausted
    GRACEFUL_FAIL --> [*]

    AI_SCORING --> AI_SKIP : Vertex AI unavailable
    AI_SKIP --> SCORE_AGGREGATION : Use rule score only
```

---

## 4. LangGraph State Machine — Recruiter Screening Orchestrator (Agent 2)

> State machine governing the recruiter candidate-screening pipeline.

```mermaid
stateDiagram-v2
    [*] --> IDLE

    IDLE --> PROFILE_LOADING : Recruiter initiates screening

    PROFILE_LOADING --> RUBRIC_PARSING : Screening profile loaded
    note right of PROFILE_LOADING
        Load job requirements,
        competencies, target bands
    end note

    RUBRIC_PARSING --> QUESTION_SELECTION : Rubric parsed to criteria
    note right of RUBRIC_PARSING
        Extract CEFR/IELTS targets
        per job competency
    end note

    QUESTION_SELECTION --> CANDIDATE_INVITATION : Job-specific questions selected
    note right of QUESTION_SELECTION
        Agent 3 — Question Selector
        Maps rubric criteria to
        question bank tags
    end note

    CANDIDATE_INVITATION --> CANDIDATE_RECORDING : Candidate accepts & records
    note right of CANDIDATE_INVITATION
        Email link sent to candidate
        Time-limited assessment window
    end note

    CANDIDATE_RECORDING --> SPEECH_PROCESSING : Audio submitted
    SPEECH_PROCESSING --> RULE_SCORING : Transcript + features ready
    RULE_SCORING --> AI_SCORING : Rule scores computed
    AI_SCORING --> RUBRIC_MAPPING : All scores ready

    RUBRIC_MAPPING --> RECOMMENDATION_GENERATION : Score ↔ Rubric mapped
    note right of RUBRIC_MAPPING
        Evaluate: Does candidate meet
        each rubric criterion?
        Produce PASS / CONDITIONAL / FAIL per criterion
    end note

    RECOMMENDATION_GENERATION --> ATS_EXPORT : Recommendation ready
    note right of RECOMMENDATION_GENERATION
        Auto-recommend: Proceed / Hold / Reject
        Based on rubric compliance score
    end note

    ATS_EXPORT --> COMPLETE : Results pushed to ATS
    note right of ATS_EXPORT
        POST to Greenhouse / Lever / Workday
        via REST API with signed credentials
    end note

    COMPLETE --> [*]

    CANDIDATE_INVITATION --> TIMEOUT : Candidate does not respond in 72h
    TIMEOUT --> EXPIRED : Mark screening as expired
    EXPIRED --> [*]
```

---

## 5. LangGraph State Machine — Risk Monitor Agent (Agent 9)

> Background agent running on a daily cron schedule.

```mermaid
stateDiagram-v2
    [*] --> SCHEDULED_TRIGGER

    SCHEDULED_TRIGGER --> FETCH_ALL_ACTIVE_LEARNERS : Cron fires (daily)

    FETCH_ALL_ACTIVE_LEARNERS --> ANALYSE_PER_LEARNER : Learner list retrieved

    ANALYSE_PER_LEARNER --> CALCULATE_STAGNATION : Per learner: fetch last 30 days of scores
    CALCULATE_STAGNATION --> CALCULATE_DECLINE : Compute score delta across attempts

    CALCULATE_DECLINE --> RISK_CLASSIFICATION : Compare against thresholds
    note right of RISK_CLASSIFICATION
        STAGNATION: No improvement
        over 3 consecutive assessments
        DECLINE: Score drop greater than 10%
        OK: Normal progression
    end note

    RISK_CLASSIFICATION --> FLAG_AT_RISK : Risk detected
    RISK_CLASSIFICATION --> NO_ACTION : No risk
    RISK_CLASSIFICATION --> ANALYSE_PER_LEARNER : Next learner

    FLAG_AT_RISK --> WRITE_FLAG_TO_DB : Write at-risk record
    WRITE_FLAG_TO_DB --> NOTIFY_TEACHER : Teacher found for learner's class
    WRITE_FLAG_TO_DB --> NOTIFY_PLATFORM : Learner not in a class

    NOTIFY_TEACHER --> ANALYSE_PER_LEARNER : Continue to next learner
    NOTIFY_PLATFORM --> ANALYSE_PER_LEARNER : Continue to next learner
    NO_ACTION --> ANALYSE_PER_LEARNER : Continue to next learner

    ANALYSE_PER_LEARNER --> COMPLETE : All learners processed

    COMPLETE --> [*]
```

---

## 6. Inter-Agent Communication & Data Flow

```mermaid
sequenceDiagram
    actor Learner
    participant ORC1 as Assessment Orchestrator<br/>(Agent 1)
    participant SA3 as Question Selector<br/>(Agent 3)
    participant SA4 as Speech Analysis<br/>(Agent 4)
    participant SA5 as Rule Evaluator<br/>(Agent 5)
    participant SA6 as AI Coherence<br/>(Agent 6)
    participant SA7 as Feedback Composer<br/>(Agent 7)
    participant SA8 as Pathway Agent<br/>(Agent 8)
    participant LS as LangSmith
    participant DB as PostgreSQL

    Learner->>ORC1: start_assessment(user_id, goal, pathway)
    ORC1->>LS: trace_start(run_id)

    ORC1->>SA3: select_questions(level, pathway, history)
    SA3->>DB: query_question_bank(filters)
    DB-->>SA3: question_set[3]
    SA3-->>ORC1: questions[Part1, Part2, Part3]

    ORC1-->>Learner: present_questions(questions)
    Learner-->>ORC1: submit_audio(audio_file_ref)

    ORC1->>SA4: process_speech(audio_file_ref)
    SA4->>SA4: call_google_stt(audio)
    SA4->>SA4: extract_acoustic_features(audio)
    SA4-->>ORC1: {transcript, speech_rate, pauses, fillers}

    ORC1->>SA5: evaluate_rules(transcript, acoustic_metrics)
    SA5->>SA5: score_grammar(transcript)
    SA5->>SA5: score_lexical_diversity(transcript)
    SA5->>SA5: score_fluency(acoustic_metrics)
    SA5-->>ORC1: rule_scores{grammar, lexical, fluency, composite}

    ORC1->>SA6: evaluate_coherence(transcript, question_context)
    SA6->>SA6: call_vertex_ai(prompt_1) [call 1 of 2]
    SA6->>SA6: call_vertex_ai(prompt_2) [call 2 of 2]
    SA6->>DB: log_api_cost(cost_usd)
    SA6-->>ORC1: ai_scores{coherence, relevance, cost_usd}

    ORC1->>ORC1: aggregate_scores(rule_scores, ai_scores)
    ORC1->>DB: store_assessment_result(user_id, scores, pathway)

    ORC1->>SA7: compose_feedback(transcript, scores, questions)
    SA7->>SA7: generate_corrected_transcript()
    SA7->>SA7: generate_grammar_explanations()
    SA7->>SA7: generate_model_answer() [uses 1 of 2 AI calls]
    SA7-->>ORC1: feedback_payload{transcript, corrections, model_answer, charts}

    ORC1->>SA8: generate_pathway(user_id, weakness_profile, goals)
    SA8->>DB: query_activity_bank(level, skills, history)
    DB-->>SA8: matching_activities[]
    SA8->>SA8: build_weekly_roadmap(activities)
    SA8-->>ORC1: weekly_roadmap[7 days]

    ORC1->>DB: store_feedback(user_id, feedback_payload)
    ORC1->>DB: store_pathway(user_id, weekly_roadmap)
    ORC1->>LS: trace_end(run_id, total_cost)
    ORC1-->>Learner: assessment_complete(feedback, roadmap)
```

---

## 7. Agent Tool Registry

> Complete list of tools available to each agent.

```mermaid
graph LR
    subgraph Agent3["Agent 3 — Question Selector"]
        T3A[query_question_bank]
        T3B[filter_by_cefr_level]
        T3C[filter_by_ielts_band]
        T3D[filter_by_topic_tag]
        T3E[exclude_used_questions]
        T3F[randomise_equivalent_form]
    end

    subgraph Agent4["Agent 4 — Speech Analysis"]
        T4A[call_google_cloud_stt]
        T4B[extract_speech_rate]
        T4C[extract_pause_patterns]
        T4D[count_filler_words]
        T4E[normalise_transcript_text]
    end

    subgraph Agent5["Agent 5 — Rule Evaluator"]
        T5A[score_grammar_errors]
        T5B[score_sentence_complexity]
        T5C[score_lexical_diversity_ttr]
        T5D[score_speech_rate_wpm]
        T5E[score_pause_frequency]
        T5F[score_filler_ratio]
        T5G[map_composite_to_cefr]
        T5H[map_composite_to_ielts_band]
    end

    subgraph Agent6["Agent 6 — AI Coherence Evaluator"]
        T6A[call_vertex_ai_coherence_score]
        T6B[call_vertex_ai_relevance_score]
        T6C[check_api_call_budget]
        T6D[log_api_cost_to_db]
    end

    subgraph Agent7["Agent 7 — Feedback Composer"]
        T7A[generate_corrected_transcript]
        T7B[annotate_grammar_errors]
        T7C[suggest_vocabulary_alternatives]
        T7D[generate_model_answer_llm]
        T7E[build_metric_chart_data]
        T7F[store_feedback_to_db]
    end

    subgraph Agent8["Agent 8 — Learning Pathway"]
        T8A[analyse_weakness_dimensions]
        T8B[query_activity_bank_by_tags]
        T8C[filter_activity_history]
        T8D[prioritise_by_goal_timeline]
        T8E[build_weekly_day_plan]
        T8F[store_pathway_to_db]
    end

    subgraph Agent9["Agent 9 — Risk Monitor"]
        T9A[fetch_learner_score_history]
        T9B[calculate_score_delta]
        T9C[detect_stagnation_pattern]
        T9D[detect_decline_pattern]
        T9E[write_at_risk_flag_to_db]
        T9F[send_teacher_alert_notification]
        T9G[send_learner_in_app_alert]
    end
```

---

## 8. LangSmith Observability Coverage

> Every agent run is traced through LangSmith for cost tracking, debugging, and research audit.

```mermaid
graph TD
    subgraph TracedAgents["All Agent Runs → LangSmith"]
        ORC1_T[Assessment Orchestrator Trace]
        ORC2_T[Recruiter Orchestrator Trace]
        ORC9_T[Risk Monitor Trace]
        SA3_T[Question Selector Trace]
        SA4_T[Speech Analysis Trace]
        SA5_T[Rule Evaluator Trace]
        SA6_T[AI Coherence Trace]
        SA7_T[Feedback Composer Trace]
        SA8_T[Pathway Agent Trace]
    end

    subgraph LangSmithData["LangSmith Captured Data"]
        LS_RUN[Run ID & Timestamps]
        LS_COST[API Cost per Run\nVertex AI USD]
        LS_TOKENS[Token Count per AI Call]
        LS_ERRORS[Error Logs & Retries]
        LS_LATENCY[Latency per Agent Node]
        LS_INPUTS[Inputs & Outputs per Node]
    end

    subgraph Consumers["Data Consumers"]
        ADMIN_DASH[Admin Cost Dashboard]
        RESEARCH_EXPORT[Research Anonymized Export]
        ALERT_SYS[Alert System\nLatency / Error Spikes]
    end

    ORC1_T --> LS_RUN
    ORC1_T --> LS_COST
    SA6_T --> LS_TOKENS
    SA6_T --> LS_COST
    SA7_T --> LS_TOKENS
    SA4_T --> LS_LATENCY
    ORC1_T --> LS_ERRORS
    ORC2_T --> LS_INPUTS

    LS_COST --> ADMIN_DASH
    LS_RUN --> RESEARCH_EXPORT
    LS_ERRORS --> ALERT_SYS
    LS_LATENCY --> ALERT_SYS

    style TracedAgents fill:#fce7f3,stroke:#ec4899
    style LangSmithData fill:#dbeafe,stroke:#3b82f6
    style Consumers fill:#dcfce7,stroke:#22c55e
```

---

## 9. Technology Stack per Agent

| Agent | Framework | LLM/AI Backend | Tools / Libraries | Trigger |
|-------|-----------|---------------|-------------------|---------|
| Assessment Orchestrator | LangGraph `StateGraph` | — | LangChain, LangSmith | Routed by Classifier Agent |
| Recruiter Orchestrator | LangGraph `StateGraph` | — | LangChain, LangSmith | Routed by Classifier Agent |
| Question Selector | LangChain `Tool` | — | PostgreSQL client, pandas | Called by orchestrators |
| Speech Analysis | LangChain `Tool` | Google Cloud STT | `google-cloud-speech`, `librosa` | Called by orchestrators |
| Rule-Based Evaluator | LangChain `Tool` | — | `language_tool_python`, `nltk`, custom rules | Called by orchestrators |
| AI Coherence Evaluator | LangChain `Tool` | Google Vertex AI (Gemini) | `google-cloud-aiplatform`, budget guard | Called by orchestrators |
| Feedback Composer | LangChain `Tool` | Google Vertex AI (Gemini) | Template engine, diff library | Called by orchestrators |
| Learning Pathway Agent | LangChain `Tool` | — | PostgreSQL client, rule engine | Called by orchestrators |
| Risk Monitor Agent | LangGraph `StateGraph` | — | PostgreSQL client, notification client | Cron / `APScheduler` |
| **Classifier Agent** | LangChain `Tool` + FastAPI middleware | — | PyJWT, intent classifier, Redis client | Every inbound HTTP request |
| **Context Manager Agent** | LangChain `Tool` | — | `redis-py`, SQLAlchemy, PostgreSQL client | Called by every orchestrator at start/end |

---

## 10. Classifier Agent — Detailed Design (Agent 10)

### Role in the System

The Classifier Agent is the **single entry point** for every request entering the agent layer. It acts as the intelligent router that inspects every inbound HTTP request and produces a `ClassifiedRequest` object consumed by the correct orchestrator. Without it, each orchestrator would need to independently parse user identity, determine intent, and apply routing logic — duplicating logic and creating inconsistency.

### What It Classifies

| Dimension | Values | Source |
|-----------|--------|--------|
| **User Role** | Learner, Teacher, Recruiter, Admin, Research Analyst | JWT claims (`role` field) |
| **Intent** | `start_assessment`, `view_dashboard`, `initiate_screening`, `join_class`, `export_data`, `configure_bank` | Request path + body |
| **Session Type** | `new_assessment`, `reassessment`, `recruiter_screen`, `teacher_review`, `admin_action` | User history + request type |
| **Learning Pathway** | `CEFR`, `IELTS` | Learner profile goal (pre-loaded from DB) |
| **Orchestrator Target** | `AssessmentOrchestrator`, `RecruiterOrchestrator`, `AdminService`, `TeacherService` | Derived from role + intent |

### Tool Registry

| Tool | Responsibility |
|------|---------------|
| `decode_jwt_claims` | Extract `user_id`, `role`, `institution_id` from the signed JWT |
| `classify_user_role` | Validate and normalise role to one of the 5 known user types |
| `classify_intent` | Match request path + action field to a known intent enum |
| `classify_session_type` | Cross-reference intent with user history to determine session type (new vs. re-assessment) |
| `classify_learning_pathway` | Fetch learner goal from DB/Redis cache and return `CEFR` or `IELTS` |
| `route_to_orchestrator` | Emit the `ClassifiedRequest` to the target orchestrator via internal event bus |
| `return_clarification_request` | If confidence < 0.85, return a structured question to the frontend to resolve ambiguity |

### Classification Flow

```mermaid
flowchart TD
    REQ([Inbound HTTP Request]) --> DECODE[decode_jwt_claims\nExtract user_id, role]

    DECODE --> ROLE_CHECK{Role valid?}
    ROLE_CHECK -->|No| REJECT[Return 401 Unauthorized]
    ROLE_CHECK -->|Yes| CLASSIFY_INTENT[classify_intent\nMatch path + body to intent enum]

    CLASSIFY_INTENT --> INTENT_CONF{Intent confidence\ngreater than 0.85?}
    INTENT_CONF -->|No| CLARIFY[return_clarification_request\nAsk frontend for disambiguation]
    CLARIFY --> REQ

    INTENT_CONF -->|Yes| CLASSIFY_SESSION[classify_session_type\nNew vs. reassessment?]
    CLASSIFY_SESSION --> PATHWAY{Role = Learner?}

    PATHWAY -->|Yes| CLASSIFY_PATHWAY[classify_learning_pathway\nLoad goal from Redis or DB]
    PATHWAY -->|No| SKIP_PATHWAY[Pathway = N/A]

    CLASSIFY_PATHWAY --> BUILD_OBJ[Build ClassifiedRequest\nuser_id, role, intent, session_type, pathway, target]
    SKIP_PATHWAY --> BUILD_OBJ

    BUILD_OBJ --> ROUTE{Orchestrator Target?}
    ROUTE -->|AssessmentOrchestrator| ORC1[Route to Agent 1\nAssessment Orchestrator]
    ROUTE -->|RecruiterOrchestrator| ORC2[Route to Agent 2\nRecruiter Orchestrator]
    ROUTE -->|TeacherService| SVC_T[Route to Teacher\nDashboard Service]
    ROUTE -->|AdminService| SVC_A[Route to Admin\nManagement Service]

    style REJECT fill:#ffe4e6,stroke:#f87171
    style CLARIFY fill:#fef9c3,stroke:#eab308
    style BUILD_OBJ fill:#dcfce7,stroke:#22c55e
```

### ClassifiedRequest Object

```typescript
ClassifiedRequest {
    user_id:             str          // from JWT
    role:                RoleEnum     // Learner | Teacher | Recruiter | Admin | ResearchAnalyst
    intent:              IntentEnum   // start_assessment | initiate_screening | view_dashboard | ...
    session_type:        SessionEnum  // new_assessment | reassessment | recruiter_screen | ...
    pathway:             PathwayEnum  // CEFR | IELTS | N/A
    orchestrator_target: str          // AssessmentOrchestrator | RecruiterOrchestrator | ...
    confidence:          float        // classification confidence score
    timestamp:           datetime
}
```

### Why It Matters

- **Separation of concerns**: Orchestrators receive a fully labelled context and never contain routing logic
- **Single responsibility**: All role/intent/pathway classification lives in one agent, making it easy to update without touching orchestrators
- **Auditable routing**: Every routing decision is logged to LangSmith with the confidence score, enabling debugging of misrouted sessions
- **Extensibility**: Adding a new user role or intent only requires updating the Classifier Agent, not all orchestrators

---

## 11. Context Manager Agent — Detailed Design (Agent 11)

### Role in the System

The Context Manager Agent is the **shared memory layer** for the entire agent network. It ensures that every orchestrator always starts a run with a complete, accurate picture of the user's history and active session — and ends a run by persisting the updated state. It prevents data loss on session interruption and keeps LLM context windows within budget by trimming history to the relevant window.

### Two-Tier Memory Architecture

| Tier | Backend | Data Stored | TTL / Retention |
|------|---------|-------------|-----------------|
| **Short-term (Session)** | Redis | In-flight `SessionState` (current step, partial scores, audio job ID) | 24 hours from last write |
| **Long-term (Profile)** | PostgreSQL | `UserContextProfile` (full assessment history, weakness profile, pathway history, activity log) | Indefinite (audit-grade) |

### What It Stores Per User

**`SessionState`** (Redis — per active session):
```typescript
SessionState {
    session_id:      str
    user_id:         str
    orchestrator:    str           // which orchestrator owns this session
    current_step:    str           // last completed LangGraph node
    partial_scores:  dict          // scores computed so far (for resume)
    audio_job_id:    str | null    // async STT job ID if in-flight
    created_at:      datetime
    last_updated:    datetime
}
```

**`UserContextProfile`** (PostgreSQL — per user, always up to date):
```typescript
UserContextProfile {
    user_id:              str
    role:                 RoleEnum
    pathway:              PathwayEnum
    current_level:        str       // CEFR sublevel or IELTS band
    weakness_profile:     dict      // {grammar: 0.6, lexical: 0.8, fluency: 0.5, ...}
    assessment_history:   list      // last N assessments (trimmed to 3 for LLM context)
    activity_log:         list      // completed activities (last 4 weeks)
    improvement_index:    float     // composite progress metric
    last_assessment_date: datetime
}
```

### Tool Registry

| Tool | Responsibility |
|------|---------------|
| `store_session_context` | Write/update `SessionState` in Redis with a 24h TTL |
| `retrieve_session_context` | Fetch `SessionState` from Redis by `session_id`; return `null` if expired |
| `clear_session` | Delete `SessionState` from Redis after successful completion |
| `get_user_context_profile` | Fetch full `UserContextProfile` from PostgreSQL |
| `update_weakness_profile` | Merge new scoring results into the user's weakness dimension weights |
| `append_assessment_to_history` | Add a new assessment record to the user's longitudinal history |
| `trim_context_for_llm` | Return only the last 3 assessments + current weakness profile (token-budget safe) |
| `prune_stale_sessions` | Background cleanup: delete Redis keys older than TTL |

### Context Read/Write Flow

```mermaid
sequenceDiagram
    participant CLS as Classifier Agent<br/>(Agent 10)
    participant ORC as Any Orchestrator<br/>(Agent 1 or 2)
    participant CTX as Context Manager Agent<br/>(Agent 11)
    participant REDIS as Redis Session Cache
    participant DB as PostgreSQL User Profiles

    CLS->>ORC: ClassifiedRequest{user_id, role, intent, ...}

    Note over ORC,CTX: ON SESSION START — Context Read
    ORC->>CTX: get_user_context_profile(user_id)
    CTX->>REDIS: GET session_id
    REDIS-->>CTX: SessionState (or null if new)
    CTX->>DB: SELECT from user_context WHERE user_id
    DB-->>CTX: UserContextProfile
    CTX->>CTX: trim_context_for_llm(profile, max_assessments=3)
    CTX-->>ORC: SessionState + UserContextProfile

    Note over ORC: Orchestrator runs its state machine using enriched context

    ORC->>ORC: Execute sub-agents 3 through 8

    Note over ORC,CTX: AFTER EACH STEP — Partial State Write
    ORC->>CTX: store_session_context(session_id, partial_state)
    CTX->>REDIS: SET session_id partial_state EX 86400
    REDIS-->>CTX: OK

    Note over ORC,CTX: ON SESSION COMPLETE — Profile Update
    ORC->>CTX: append_assessment_to_history(user_id, result)
    CTX->>DB: INSERT INTO assessment_history
    ORC->>CTX: update_weakness_profile(user_id, new_scores)
    CTX->>DB: UPDATE user_context SET weakness_profile
    ORC->>CTX: clear_session(session_id)
    CTX->>REDIS: DEL session_id
    CTX-->>ORC: Profile updated, session cleared
```

### Session Resume Flow (Interrupted Assessment)

```mermaid
flowchart TD
    START([Learner returns after interruption]) --> CLS_CLASSIFY[Classifier Agent\nclassifies as session_resume]
    CLS_CLASSIFY --> ORC_START[Assessment Orchestrator\ncalls get_user_context_profile]
    ORC_START --> REDIS_CHECK{SessionState\nexists in Redis?}

    REDIS_CHECK -->|Yes — within 24h| LOAD_STATE[Load partial_state\nfrom Redis]
    LOAD_STATE --> RESUME_STEP[Resume LangGraph at\nlast_completed_step]
    RESUME_STEP --> CONTINUE[Continue from where\nlearner left off]

    REDIS_CHECK -->|No — expired or new| LOAD_PROFILE[Load UserContextProfile\nfrom PostgreSQL only]
    LOAD_PROFILE --> START_FRESH[Start fresh assessment\nwith full historical context]

    CONTINUE --> COMPLETE([Assessment completes normally])
    START_FRESH --> COMPLETE

    style LOAD_STATE fill:#dcfce7,stroke:#22c55e
    style LOAD_PROFILE fill:#fef9c3,stroke:#eab308
```

### Why It Matters

- **Resilience**: A network drop or browser refresh does not lose a learner's in-progress assessment — they resume from the last completed step
- **Context quality**: Orchestrators receive a rich, pre-assembled context object instead of querying 4+ tables themselves
- **Token budget enforcement**: The `trim_context_for_llm` function ensures LLM-backed agents (6 and 7) never exceed the Vertex AI context window, keeping AI call costs predictable
- **Longitudinal accuracy**: Because every session end writes to `UserContextProfile`, the weakness profile and improvement index are always up to date for the Risk Monitor Agent, Learning Pathway Agent, and Teacher Dashboard
- **Single source of truth**: All agents read user context from one place; no stale data from concurrent reads

---

## 12. Cost Control Strategy

A critical business requirement is keeping AI costs to ≤2 Vertex AI calls per assessment. The following enforcement mechanism is built into Agent 6 and Agent 7:

```mermaid
flowchart TD
    START([Assessment scoring begins]) --> CHECK_BUDGET{API call budget\nremaining?}

    CHECK_BUDGET -->|Budget = 2| USE_FOR_COHERENCE[Agent 6: Call Vertex AI\nfor Coherence Score\nCall #1 of 2]
    USE_FOR_COHERENCE --> DECREMENT1[Budget = 1]
    DECREMENT1 --> USE_FOR_RELEVANCE[Agent 6: Call Vertex AI\nfor Relevance Score\nCall #2 of 2]
    USE_FOR_RELEVANCE --> DECREMENT2[Budget = 0]

    DECREMENT2 --> FEEDBACK_STEP{Model Answer\nneeded?}
    FEEDBACK_STEP -->|Yes, budget = 0| USE_TEMPLATE[Agent 7: Generate Model Answer\nfrom Rule Template\nNo AI call]
    FEEDBACK_STEP -->|Pre-budget option| AI_MODEL_ANSWER[Agent 7: Use 1 remaining\ncall for Model Answer]

    CHECK_BUDGET -->|Budget = 0| SKIP_AI[Skip AI scoring\nUse rule scores only\nFlag in result]
    SKIP_AI --> RULE_ONLY[Return rule-based\nscore composite]

    USE_TEMPLATE --> LOG_COST[Log $0 for template path]
    RULE_ONLY --> LOG_COST
    USE_FOR_RELEVANCE --> LOG_AI_COST[Log AI cost to DB\nvia LangSmith]

    LOG_COST --> DONE([Scoring complete])
    LOG_AI_COST --> DONE

    style CHECK_BUDGET fill:#fef9c3,stroke:#eab308
    style SKIP_AI fill:#ffe4e6,stroke:#f87171
    style LOG_AI_COST fill:#dcfce7,stroke:#22c55e
```

---

## 13. Summary
|----------|-------|-------|
| **Orchestrators** | 2 | Assessment Orchestrator, Recruiter Screening Orchestrator |
| **Sub-Agents** | 6 | Question Selector, Speech Analysis, Rule-Based Evaluator, AI Coherence Evaluator, Feedback Composer, Learning Pathway Agent |
| **Background Agents** | 1 | Risk Monitor Agent |
| **Infrastructure Agents** | 2 | Classifier Agent, Context Manager Agent |
| **Total** | **11** | |

### Implementation Order (Recommended)
1. **Agent 11** (Context Manager) — build the Redis + PostgreSQL memory layer first; everything depends on it
2. **Agent 10** (Classifier) — build the routing gate so all requests are properly labelled before reaching orchestrators
3. **Agent 4** (Speech Analysis) — foundation; everything depends on transcripts
4. **Agent 5** (Rule-Based Evaluator) — core scoring; no external dependencies
5. **Agent 3** (Question Selector) — enables assessments to run
6. **Agent 1** (Assessment Orchestrator) — wires Agents 10, 11, 3, 4, 5 together; MVP-ready
7. **Agent 6** (AI Coherence Evaluator) — adds AI layer on top of rule scores
8. **Agent 7** (Feedback Composer) — completes the learner feedback loop
9. **Agent 8** (Learning Pathway Agent) — personalised pathway generation
10. **Agent 2** (Recruiter Screening Orchestrator) — recruiter flow
11. **Agent 9** (Risk Monitor Agent) — background monitoring; can be added last
