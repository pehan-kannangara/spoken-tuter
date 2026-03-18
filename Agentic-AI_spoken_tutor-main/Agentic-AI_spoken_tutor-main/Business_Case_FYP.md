# Business Case Template

## 1. Executive Summary

This business case proposes the development of a web-based,
mobile-responsive intelligent decision support system for early disease
risk detection in companion animals. The system leverages Machine
Learning for predictive risk classification and an Agentic AI reasoning
layer to generate contextual, explainable preventive recommendations.

Pet owners input symptom and behavioral observations through structured
forms or an AI-assisted chatbot interface. The system processes this
data through a cloud-based ML inference service to classify disease risk
into Low, Medium, or High categories.

Based on risk level, the Agentic AI layer autonomously generates
actionable guidance including home care advice, veterinary consultation
recommendations (24--48 hour thresholds), vaccination alerts, and
emergency escalation notifications.

The platform integrates geolocation-based veterinary clinic discovery
using Google Maps API and cross-references predicted disease types with
clinic specialization and appointment availability stored within
Firebase Firestore.

The proposed solution enhances early intervention, reduces treatment
costs, supports preventive veterinary care, and establishes a scalable
AI-driven digital health platform for the Sri Lankan pet healthcare
ecosystem.

------------------------------------------------------------------------

## 2. Introduction and Summary

### a. Rationale for Preferred Option

The preferred option is to develop a web-based mobile AI decision
support system using Machine Learning and Agentic AI, rather than a
purely informational app or a clinician-only tool.

This option is selected because it:

-   Enables early disease risk prediction using symptom data.
-   Provides autonomous reasoning and guidance.
-   Integrates vaccination tracking, emergency alerts, and real-time
    veterinary discovery.
-   Is accessible to pet owners, not limited to veterinarians.
-   Aligns directly with modern healthcare trends in AI-driven
    preventive care.

Alternative options such as static information systems or vet-only tools
were rejected due to their lack of autonomy, real-time risk prediction,
and owner engagement.

### b. Current Business Process

Pet owners typically rely on: - Internet searches or informal advice -
Delayed veterinary visits after symptoms worsen - Manual tracking of
vaccinations and health history

Veterinary healthcare is largely reactive rather than preventive.
Existing systems: - Focus on appointments, reminders, or clinical
diagnostics - Do not offer early risk prediction or autonomous guidance
at the owner level

There is no widely adopted mobile system that provides AI-driven,
personalized, preventive decision support for pet owners.

### c. Description of the Problem

Companion animal health management relies heavily on pet owners' ability
to recognize early symptoms and seek timely veterinary care. However,
early-stage diseases often present with non-specific behavioral or
physiological changes such as reduced appetite, lethargy, or altered
urination patterns, which are difficult for non-experts to interpret.

Existing digital pet care applications primarily focus on vaccination
reminders, health record storage, or appointment booking and do not
provide intelligent early risk assessment or preventive decision
support.

The absence of an accessible, data-driven system for early disease risk
identification leads to late diagnosis, increased treatment costs,
reduced recovery outcomes, and compromised animal welfare.

### d. Opportunity

The increasing number of companion animal owners and growing awareness
of preventive pet healthcare present a significant opportunity for a
digital early disease risk assessment solution.

Business opportunities include: - Growing market demand for digital pet
health and telehealth solutions - Potential integration with veterinary
clinics and pet care providers - Development of a scalable mobile health
platform - Future expansion into subscription-based services and
teleconsultation ecosystems

### e. Objectives

#### Primary Objective

To design and implement a scalable, AI-driven, web-based decision
support architecture integrating machine learning prediction and agentic
reasoning for early disease risk identification.

#### Specific Objectives

-   Provide an easy-to-use platform for recording symptoms and
    behavioral changes
-   Generate early disease risk predictions using machine learning
-   Deliver actionable preventive recommendations through an Agentic AI
    layer
-   Support veterinary consultation through location-based clinic
    discovery
-   Improve preventive healthcare through vaccination tracking and
    reminders
-   Maintain centralized digital health history
-   Increase early awareness through proactive alerts
-   Ensure accessibility through a mobile-responsive interface

------------------------------------------------------------------------

## 3. Project Scope

### System Users

-   Dog and cat owners
-   Veterinary clinics (information listing and availability)

### Core Functional Scope

-   User registration and authentication
-   Pet profile creation
-   Symptom and behavioral data entry
-   Machine learning-based risk prediction
-   Agentic AI-based preventive guidance
-   Vaccination tracking and reminders
-   Emergency alerts
-   Veterinary clinic discovery
-   Health history storage
-   Notification system

### Technology Stack

-   Frontend: React.js with Tailwind CSS
-   Backend: Node.js with Express.js
-   Database: Firebase Firestore
-   ML Service: Python FastAPI with Google Vertex AI
-   Agentic AI: LangChain, LangGraph, LangSmith
-   Maps Integration: Google Maps API
-   Notifications: Firebase Cloud Messaging

------------------------------------------------------------------------

## 4. Implementation Plan

  ------------------------------------------------------------------------
  Phase      Key Activities          Deliverables          Duration
  ---------- ----------------------- --------------------- ---------------
  Phase 1    Requirement Analysis &  Approved BRD          2 Weeks
             Architecture                                  

  Phase 2    ML Development          Trained Model         4 Weeks

  Phase 3    Agentic AI Development  Functional Agent      3 Weeks
                                     Module                

  Phase 4    Application Development Working Web           4 Weeks
                                     Application           

  Phase 5    Testing & QA            Validated System      2 Weeks

  Phase 6    Pilot Deployment        Pilot Report          2 Weeks

  Phase 7    Final Deployment        Operational System    Ongoing
  ------------------------------------------------------------------------

------------------------------------------------------------------------

## 5. Risk Management Plan

  Risk ID   Description                    Likelihood   Impact   Level
  --------- ------------------------------ ------------ -------- --------
  R1        Low-quality ML training data   Medium       High     High
  R2        Incorrect risk predictions     Medium       High     High
  R3        Data privacy breach            Low          High     High
  R4        API integration issues         Medium       Medium   Medium
  R5        Low user adoption              Medium       Medium   Medium
  R6        Ethical concerns               Low          High     Medium

### Mitigation Strategies

-   Use validated datasets and preprocessing
-   Implement explainable AI outputs
-   Secure authentication and encryption
-   Conduct Agile-based integration testing
-   Perform usability testing
-   Include clear disclaimers

------------------------------------------------------------------------

## 6. Conclusion and Recommendations

The proposed Agentic AI-driven decision support system addresses a clear
gap in preventive companion animal healthcare by integrating machine
learning-based early risk prediction, autonomous reasoning, and
preventive care management within a mobile-accessible platform.

### Recommendations

-   Follow Agile phased implementation
-   Prioritize high-quality dataset preparation
-   Ensure strong data privacy compliance
-   Position the system strictly as a preventive decision support tool
-   Conduct pilot testing before large-scale deployment
