# Requirement Analysis for AI-Powered Adaptive Spoken English Learning Platform

## 1. Overview

This document captures the requirements for a full-stack adaptive spoken English learning platform inspired by smalltalk2me.ai, redesigned for secondary and tertiary learners and academic contexts. The system emphasizes cost-efficient AI usage, transparent rule-based logic, and structured evaluation for research validation. It supports both CEFR-based general proficiency and IELTS exam preparation paths, integrates teacher monitoring, provides light gamification, and offers value for recruiters and talent acquisition teams.

## 2. Stakeholders

- **Students**: Secondary/tertiary learners seeking to improve spoken English or prepare for IELTS.
- **Teachers/Institutional Instructors**: Create classes, assign tasks, monitor student progress, intervene when needed.
- **Recruiters/Talent Acquisition**: Use platform output to screen or evaluate candidates.
- **Curriculum Designers**: Configure question banks, activity banks, and adaptation rules.
- **Administrators**: Manage user accounts, system settings, access control.
- **Research Analysts**: Extract and analyze data for validation studies.
- **System Integrators/DevOps**: Maintain infrastructure, APIs, and integrations.

## 3. Actor & Use Case Summary

- User registration & authentication (students, teachers).
- Teachers create classes and share codes.
- Students join classes or use independently.
- Profile collection for learners (education, exposure, goals).
- Pathway selection (CEFR or IELTS).
- Initial skill assessment and subsequent reassessments.
- Speech recording & transcription.
- Hybrid scoring engine evaluation.
- Feedback generation with corrections and explanations.
- Personalized learning path recommendations.
- Activity completion tracking and visual progress.
- Gamification (streaks, badges, milestones).
- Teacher dashboards and analytics with risk flagging.
- Recruiter screening workflows and API integration.
- Data export for research and reporting.

## 4. Functional Requirements

### 4.1. Authentication & Authorization

1. Secure role-based registration for students and teachers.
2. Teachers can create/terminate classes, generate unique codes.
3. Students can join classes via code or use independently.
4. Access control ensures students access learning tools; teachers access analytics.

### 4.2. Learner Profiling

1. Collect: education level, English exposure, confidence, academic field, target IELTS band (if applicable), target timeline.
2. Process profile with rule-based logic to adjust pacing/focus areas.

### 4.3. Assessment Module

1. Use categorized question bank mapped by CEFR levels and IELTS bands.
2. Conduct initial skill assessments: short personal questions, long-turn response, analytical discussion.
3. Store recordings and transcriptions securely.
4. Reassessment selects new equivalent questions.

### 4.4. Scoring Engine

1. Rule-based extraction of objective features: grammar errors, sentence complexity, lexical diversity, speech rate, pauses, filler words.
2. Limited AI for coherence and relevance scoring.
3. Map scores to IELTS band or CEFR levels.
4. Compute improvement metrics and Improvement Index across attempts.
5. Maintain comparability with human rater scores for research.

### 4.5. Feedback Module

1. Display original transcript and corrected version.
2. Provide grammar explanations and vocabulary suggestions.
3. Show improved model answer and pronunciation observations.
4. Visualize metrics: speed, filler words, lexical diversity.

### 4.6. Personalized Learning Pathway

1. Analyze weakest skill dimensions, level, timeline, and goals.
2. Generate weekly roadmap using activity bank tagged by level, band, topic, and difficulty.
3. Avoid repetition using activity history.

### 4.7. Progress Tracking & Gamification

1. Visual progress bar for level/band progression.
2. Streak system for consecutive practice days.
3. Weekly consistency tracker.
4. Milestone badges and achievement notifications.
5. Optional social sharing of achievements.

### 4.8. Teacher Dashboard & Analytics

1. Class-level analytics: average level, skill distribution heatmap, improvement trends, at-risk students.
2. Individual student reports: timeline graphs, skill breakdown, activity completion.
3. Exportable reports for academic monitoring.

### 4.9. Recruiter Screening Integration

1. Screening profiles based on recruiter criteria.
2. API calls to/from TA systems for candidate data.
3. Agentic question selection and scoring for screens.
4. Provide screening status and scores to recruiter systems.

### 4.10. Data Management

1. Relational database for users, classes, transcripts, scores, activity history, improvement records, streaks, badges, risk flags.
2. Speech processing module for transcription and acoustic analysis.
3. Export facilities for research data.

### 4.11. Continuous Improvement & Monitoring

1. Automated detection of stagnation or decline using threshold rules.
2. Flag at-risk students for teacher intervention.
3. Track operational cost metrics for AI usage.

## 5. Non-functional Requirements

- **Scalability**: Support thousands of concurrent users, horizontal scaling of services.
- **Security**: GDPR/FERPA compliance, encrypted storage of audio and personal data.
- **Performance**: Low latency for assessments and feedback generation.
- **Modularity**: AI services modular and invoked asynchronously.
- **Reliability**: High availability of core services, backups for data.
- **Maintainability**: Clean code, documented APIs, configuration-driven logic.
- **Cost Efficiency**: Minimize AI calls; rule-based logic for core operations.

## 6. Data Requirements

- User profiles, authentication credentials.
- Class and enrollment records.
- Question bank metadata and content.
- Activity bank with tags and difficulty scores.
- Audio recordings, transcriptions, and analysis metrics.
- Scoring records and improvement indices.
- Badge and achievement history.
- Risk flags and teacher notes.

## 7. System Architecture Overview

1. **Frontend**: React/Angular/Vue for interactive dashboards, progress visuals, and assessment UI.
2. **Backend API**: Node.js/Python/Java with REST/GraphQL endpoints.
3. **Speech Module**: Microservice for recording, preprocessing, speech-to-text, acoustic feature extraction.
4. **AI Services**: Modular microservices for coherence scoring and other limited model calls.
5. **Database**: PostgreSQL/MySQL for relational data; optional NoSQL for logs.
6. **Integration Layer**: API connectors for TA databases and social sharing.
7. **Rule Engine**: Business logic service implementing adaptive pathways and scoring thresholds.

## 8. Research & Validation Support

- Facility to export anonymized datasets.
- Logging of AI vs human scoring comparisons.
- Metrics for cost-per-improvement and time-to-level.

## 9. Compliance & Ethical Considerations

- Transparent scoring logic and accessible explanations to learners.
- Optional consent for sharing data with recruiters or social networks.
- Regular audits of AI bias and fairness.

---

This requirements analysis sets the foundation for detailed design, development, and project planning. The next steps are to translate these requirements into user stories, technical specifications, and data models.