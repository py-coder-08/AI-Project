# Project

# Instruction for the NightingaleMedicalBot_sample.py
'''
1. Download LM Studio and Bionic app
2. Open Bionic app and Install Qwen 3.5 9B AI and have a test chat to initialize it
3. Go to setting -> Local Model API -> Enable Local AI Server -> Copy the Base URL -> Paste into the code
4. Run the code
5. Enter your instruction in the terminal to start chat with it
   
'''

# ATTRIBUTION
# PYTHON MODULE USED:
1. os
2. re
3. sys
4. json
5. inspect
6. sqlite3
7. datetime
8. getpass
9. unittest
10. requests
11. dataclasses
12. typing
13. rich

# TECHNICAL BRIEF
# System Architecture

┌────────────────────────────────────────────────────────────────────────┐
│                   Client Terminal (main execution loop)                │
└────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│             MedicalAgentTools.redact_phi(user_input)                  │
│       (Strips MyKad ICs, MY Phones, Emails, Name patterns)             │
└────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│        MedicalAgentTools.evaluate_emergency_risk(clean_input)         │
└────────────────────────────────────────────────────────────────────────┘
                  │                                   │
      (Risk Level: High / Med)                 (Risk Level: Low)
                  │                                   │
                  ▼                                   ▼
┌───────────────────────────────────┐ ┌──────────────────────────────────┐
│ PatientMemoryDB.create_escalation │ │ Agent.chat(clean_input)          │
│ - Generates profile snapshot      │ │ - System & Context Injection     │
│ - Writes to 'escalations' table   │ │ - Qwen 3.5 via Local API         │
│ - Logs event to 'audit_logs'      │ │ - Dynamic Tool Calling           │
└───────────────────────────────────┘ └──────────────────────────────────┘
                                                       │
                                                       ▼
                                      ┌──────────────────────────────────┐
                                      │ save_patient_record Tool         │
                                      │ - Calls add_or_update_record()   │
                                      │ - UPSERT to 'patient_records'    │
                                      │ - Logs mutation to 'audit_logs'  │
                                      └──────────────────────────────────┘

 # Core Execution Flow
1. Edge PHI Scrubbing: Raw inputs pass through MedicalAgentTools.redact_phi() prior to LLM inference, replacing Malaysian MyKad IDs, phone numbers, emails, and names with anonymized tokens.
2. Deterministic Risk Assessment: Inputs bypass LLM processing and evaluate against regex triggers via "MedicalAgentTools.evaluate_emergency_risk()". High/Medium emergency conditions bypass text generation entirely and trigger an emergency protocol.
3. Context Injection & LLM Reasoning: For low-risk queries, active patient medical records are retrieved from patient_records and dynamically injected into the system context.

# Data Schema
  ┌───────────────────────────────┐               ┌───────────────────────────────┐
  │          audit_logs           │               │        patient_records        │
  ├───────────────────────────────┤               ├───────────────────────────────┤
  │ id: INTEGER (PK)              │               │ id: INTEGER (PK)              │
  │ user_id: TEXT                 │               │ patient_id: TEXT              │◄───┐
  │ event_type: TEXT              │               │ category: TEXT [CHECK]        │    │
  │ metadata_json: TEXT           │               │ value: TEXT                   │    │
  │ timestamp: DATETIME           │               │ status: TEXT DEFAULT 'active' │    │
  └───────────────────────────────┘               │ provenance_pointer: TEXT      │─┐  │
                                                  │ audio_transcript_id: TEXT     │ │  │
                                                  │ reported_at: DATETIME         │ │  │
                                                  │ updated_at: DATETIME          │ │  │
                                                  │ UNIQUE(patient_id, cat, val)  │ │  │
                                                  └───────────────────────────────┘ │  │
                                                                                    │  │
                                                  ┌───────────────────────────────┐ │  │
                                                  │          escalations          │ │  │
                                                  ├───────────────────────────────┤ │  │
                                                  │ id: INTEGER (PK)              │ │  │
                                                  │ patient_id: TEXT              │─┼──┘
                                                  │ triggering_message: TEXT      │ │
                                                  │ triage_summary: TEXT          │ │
                                                  │ profile_snapshot: TEXT        │ │
                                                  │ provenance_pointer: TEXT      │◄┘
                                                  │ acquisition_context: TEXT     │
                                                  │ status: TEXT DEFAULT 'pending'│
                                                  │ clinician_response: TEXT      │
                                                  │ created_at: DATETIME          │
                                                  └───────────────────────────────┘

# Ethics
TO BE DETERMINED

# First Principle Thinking
1. Determinism Over Prediction for Safety. Large language models are non-deterministic pattern matchers. Emergency safety checks must rely on deterministic code (MedicalAgentTools.evaluate_emergency_risk) that executes before invoking an LLM.

# Trade-offs
1. No Automated Medical Diagnosis: Nightingale acts purely as a triage and memory extraction assistant. Issuing formal medical diagnoses or drug prescriptions was cut to comply with healthcare legislation and eliminate clinical liability.
2. Local Functionality. It only works locally and unable to do real-world action such as calling emergency number.
