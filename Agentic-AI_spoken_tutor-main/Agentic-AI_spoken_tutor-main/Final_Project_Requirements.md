# Final Project Requirements: AI-Powered Adaptive Spoken English Learning Platform

## 1. Executive Summary

This comprehensive requirements specification outlines the development of a full-stack, web-based and mobile-responsive AI-powered Adaptive Spoken English Learning Platform. Inspired by smalltalk2me.ai but optimized for secondary and tertiary learners, the platform balances quality, cost-efficiency, scalability, effectiveness, and research validity.

The platform features:
- **Role-based onboarding** with pre-signup filtering (School Student, University Student, Working Professional, Teacher, Recruiter)
- **Dual-pathway evaluation** (CEFR for general improvement/school; IELTS for exam/working/interview preparation)
- **Hybrid AI-rule scoring** combining objective metrics with selective AI inference
- **Optional teacher integration** via class codes for guided learning
- **Recruiter screening workflows** with ATS integration
- **Light professional gamification** for motivation
- **Longitudinal improvement tracking** with research validation capabilities

The system serves learners, teachers, and recruiters through a transparent, cost-controlled, scalable ecosystem.

---

## 2. Stakeholders and User Roles

### 2.1. Primary Stakeholders
- **Learners (School Students, University Students, Working Professionals)**: Core users seeking spoken English improvement
- **Teachers**: Educators managing classes and student progress
- **Recruiters/Talent Acquisition Teams**: Professionals screening candidates
- **Institutions/Schools**: Organizations adopting the platform
- **Researchers**: Academics validating effectiveness

### 2.2. User Roles and Responsibilities

| Role | Description | Key Permissions |
|------|-------------|-----------------|
| **School Student** | Secondary-level learner, often teacher-guided | Take assessments, complete activities, track progress, join classes via code |
| **University Student** | Tertiary-level learner, independent or guided | Same as School Student, with optional class joining |
| **Working Professional** | Adult learner for career advancement | Independent learning, assessment, achievement sharing |
| **Teacher** | Instructor managing student groups | Create classes, generate codes, view analytics, flag at-risk students |
| **Recruiter** | Talent acquisition professional | Create screening profiles, initiate assessments, view results, export to ATS |
| **Administrator** | System maintainer | Manage users, configure settings, export data |
| **Research Analyst** | Data validator | Access anonymized data, compare scores, measure effectiveness |

---

## 3. Functional Requirements

### 3.1. User Management and Authentication
1. **Pre-Signup Role Selection**: Users select role (School Student, University Student, Working Professional, Teacher, Recruiter) before registration
2. **Goal-Based Pathway Determination** (for learners):
   - Select goal: IELTS Exam, General Improvement, For School, Working Purpose, Interview Preparation
   - Automatic pathway assignment: IELTS goals → Band score evaluation; General/School goals → CEFR level evaluation
3. **Registration and Verification**: Email/password registration, role-specific fields, email verification
4. **Login and Session Management**: Secure authentication, role-based routing
5. **Profile Management**: Update personal info, preferences, notification settings

### 3.2. Learner Workflow
1. **Class Joining (Optional)**: Enter teacher-provided code to join class; otherwise, independent learning
2. **Learner Profiling**: Collect education level, exposure, confidence, field, timeline
3. **Assessment Taking**: Record responses to structured questions (Part 1: personal; Part 2: long-turn; Part 3: discussion)
4. **Feedback Review**: View transcript corrections, explanations, metrics, model answers
5. **Learning Pathway**: Receive weekly activity plans, complete tasks, track progress
6. **Gamification**: Earn streaks, badges, share achievements
7. **Reassessment**: Take equivalent-form assessments, view improvement metrics

### 3.3. Teacher Workflow
1. **Class Management**: Create classes, generate unique codes, manage enrollment
2. **Student Monitoring**: View class analytics, individual profiles, progress timelines
3. **Risk Flagging**: Automatic detection of stagnation/decline, manual intervention logging
4. **Reporting**: Export class/individual reports for institutional use

### 3.4. Recruiter Workflow
1. **Screening Profile Creation**: Define job requirements, competencies, rubrics
2. **Candidate Assessment**: Initiate screenings, select questions via agentic orchestration
3. **Result Review**: View scores, rubric compliance, recommendations
4. **ATS Integration**: API connections to push results back to recruitment systems

### 3.5. Assessment and Scoring
1. **Question Bank Management**: Categorized questions mapped to CEFR/IELTS levels
2. **Speech Processing**: Record audio, transcribe, extract acoustic features
3. **Hybrid Scoring**:
   - Rule-based: Grammar errors, lexical diversity, speech rate, pauses, fillers
   - AI-limited: Coherence and relevance scoring (2 API calls per assessment)
4. **Score Mapping**: Convert metrics to CEFR levels or IELTS bands
5. **Improvement Tracking**: Calculate gains, composite Improvement Index

### 3.6. Content and Personalization
1. **Activity Bank**: Tagged activities by level, skill, topic, difficulty
2. **Recommendation Engine**: Rule-based pathway generation based on weaknesses/goals
3. **Progress Visualization**: Dashboards with bars, streaks, calendars, badges

### 3.7. Data Management and Export
1. **Secure Storage**: Encrypted audio/transcripts, relational data
2. **Research Exports**: Anonymized datasets for validation studies
3. **Cost Tracking**: Log API usage, compute costs for ROI analysis

---

## 4. Non-Functional Requirements

### 4.1. Performance
- **Response Time**: Dashboard load <2 seconds; feedback generation <24 hours
- **Scalability**: Support 10,000+ concurrent users with auto-scaling
- **Availability**: 99.5% uptime, 24/7 operation

### 4.2. Security and Compliance
- **Data Protection**: GDPR/FERPA compliant, encrypted storage/transmission
- **Access Control**: RBAC, secure APIs, audit logs
- **Privacy**: User consent for data sharing, anonymized exports

### 4.3. Usability
- **Accessibility**: WCAG 2.1 compliant, mobile-responsive
- **Intuitiveness**: Clean UI, guided onboarding, minimal steps
- **Localization**: Support for English (initial), extensible to other languages

### 4.4. Reliability
- **Error Handling**: Graceful failures, user-friendly error messages
- **Backup and Recovery**: Daily backups, disaster recovery plan
- **Monitoring**: Real-time alerts for system health

### 4.5. Cost Efficiency
- **AI Usage**: Limit to 2 API calls per assessment; track costs
- **Resource Optimization**: Asynchronous processing, caching
- **Modular Design**: Independent services for easy scaling

### 4.6. Maintainability
- **Code Quality**: Clean, documented, testable code
- **Modularity**: Microservices architecture
- **Documentation**: API docs, user guides, developer guides

---

## 5. Data Requirements

### 5.1. Master Data Entities
- **Users**: ID, role, email, profile data, registration date
- **Classes**: ID, teacher_id, name, code, creation date
- **Assessments**: ID, user_id, pathway, questions, scores, timestamps
- **Activities**: ID, tags (level, skill, topic), difficulty, content
- **Badges/Streaks**: User achievements, dates
- **Screening Profiles**: Recruiter-defined criteria, rubrics

### 5.2. Transactional Data
- **Audio Recordings**: Encrypted files, metadata
- **Transcriptions**: Text, confidence scores
- **Metrics**: Objective scores, AI inferences
- **Feedback**: Corrections, explanations, visualizations
- **Progress Logs**: Activity completions, streaks, badges

### 5.3. Analytical Data
- **Improvement Indices**: Composite scores, trends
- **Cost Logs**: API calls, compute usage
- **Research Datasets**: Anonymized exports

### 5.4. Data Retention and Archival
- **Active Data**: 2 years for user data
- **Archives**: 7 years for research/compliance
- **Deletion**: User-initiated data removal

---

## 6. System Architecture Overview

### 6.1. High-Level Architecture
- **Frontend**: React/Vue.js, responsive design
- **Backend**: Node.js/Python APIs, microservices
- **Database**: PostgreSQL (relational) + Redis (cache)
- **Speech Module**: ASR service, acoustic analysis
- **AI Services**: Modular coherence/relevance scoring
- **Integration Layer**: ATS APIs, social sharing, notifications

### 6.2. Technology Stack
- **Frontend**: React.js, Tailwind CSS
- **Backend**: Node.js/Express or FastAPI
- **Database**: PostgreSQL, Redis
- **AI/ML**: Python FastAPI, Google Vertex AI or similar
- **Speech**: Google Cloud Speech-to-Text
- **Cloud**: GCP/AWS/Azure
- **DevOps**: Docker, Kubernetes, CI/CD

### 6.3. Integration Points
- **ATS Systems**: REST/GraphQL APIs (Greenhouse, Lever, Workday)
- **Social Platforms**: LinkedIn, Twitter for sharing
- **Email/SMS**: Notification services
- **Payment**: Optional for premium features

---

## 7. Business Processes Summary

### 7.1. Core Workflows
1. **Onboarding**: Role selection → Goal/pathway → Registration → Profiling → Assessment
2. **Learning Cycle**: Assessment → Feedback → Pathway → Activities → Reassessment
3. **Teacher Oversight**: Class creation → Monitoring → Flagging → Intervention
4. **Recruiter Screening**: Profile setup → Assessment initiation → Result review → Export

### 7.2. Key Processes
- Hybrid scoring with rule-based + AI
- Equivalent-form reassessment for reliability
- Risk detection via threshold rules
- Longitudinal improvement tracking

---

## 8. Novelty and Research Gap

### 8.1. Business Novelty
- Hybrid AI-rule architecture for cost transparency
- Dual CEFR/IELTS pathways in one engine
- Teacher-optional class joining via codes
- Agentic recruiter screening with low-cost integration
- Professional gamification for tertiary learners
- Longitudinal Improvement Index metric

### 8.2. Research Gap Addressed
- Longitudinal reliability of adaptive speaking assessments
- Cost-effectiveness of AI-augmented learning
- Employment outcomes from language proficiency
- Professional gamification effectiveness
- Hybrid scoring validity against human raters

---

## 9. UI/UX Requirements

### 9.1. Design Principles
- Clean, professional theme (blues/grays, accent colors)
- Mobile-first responsive design
- Intuitive navigation with role-based menus
- Consistent components: cards, buttons, forms, charts

### 9.2. Key Screens and Components
1. **Pre-Signup Role Selection**: Card-based role picker
2. **Goal Selection**: Radio buttons with pathway confirmation
3. **Registration**: Form with role-specific fields
4. **Assessment Interface**: Recording widgets, progress indicators
5. **Feedback Dashboard**: Side-by-side transcripts, metric charts
6. **Learning Pathway**: Weekly calendar with activity cards
7. **Progress Dashboard**: Progress bars, streaks, badges
8. **Teacher Analytics**: Heatmaps, timelines, student tables
9. **Recruiter Portal**: Profile forms, result tables
10. **Settings/Admin**: Configuration panels, export utilities

### 9.3. Interaction Patterns
- Guided onboarding flows
- Real-time recording feedback
- Smooth transitions between screens
- Touch-friendly mobile controls

---

## 10. Implementation Roadmap

### 10.1. Phases
1. **Requirements & Design** (3 weeks): Stakeholder validation, architecture, UI mockups
2. **Core MVP** (8 weeks): Auth, assessment, scoring, feedback
3. **Personalization & Learning** (6 weeks): Pathways, activities, gamification
4. **Teacher & Recruiter Features** (4 weeks): Dashboards, integrations
5. **Testing & Validation** (4 weeks): QA, user testing, security
6. **Pilot & Launch** (2 weeks): Deployment, monitoring

### 10.2. Milestones
- End of Phase 1: Approved specs and designs
- End of Phase 2: Functional MVP
- End of Phase 4: Full feature set
- Launch: Production deployment

---

## 11. Risk Management

### 11.1. Key Risks
- Low-quality ASR transcriptions
- AI scoring bias
- User adoption challenges
- Data privacy breaches
- Scalability bottlenecks

### 11.2. Mitigation Strategies
- Validate ASR with human checks
- Audit AI outputs regularly
- Conduct usability testing
- Implement security best practices
- Design for horizontal scaling

---

## 12. Success Criteria and KPIs

### 12.1. User Adoption
- 70% assessment completion rate
- 80% user retention after 4 weeks

### 12.2. Performance
- 99.5% uptime
- <2 second dashboard load times

### 12.3. Business Impact
- Positive improvement indices (>10% average)
- Cost per assessment < $5
- Research publications enabled

### 12.4. Quality
- 95% user satisfaction in pilots
- Valid correlation with human rater scores

---

## 13. Conclusion

This final requirements specification provides a complete blueprint for building the AI-Powered Adaptive Spoken English Learning Platform. The document integrates business processes, technical requirements, UI designs, and research validation needs into a cohesive plan.

The platform's novelty lies in its cost-efficient hybrid AI approach, flexible role-based design, and research-grade capabilities, positioning it as a leader in adaptive language learning for academic and professional contexts.

Next steps: Begin implementation with Phase 1 (Requirements & Design) and secure stakeholder approval.