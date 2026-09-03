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

Client Terminal Ingestion: Accepts raw user text input inside the primary application execution loop.
Edge PHI Redaction: Passes raw strings through MedicalAgentTools.redact_phi(), replacing Malaysian MyKad ICs, phone numbers, email addresses, and explicit name phrases with anonymized tokens before any model exposure or database writes.
Pre-Flight Risk Evaluation: Route clean strings directly into MedicalAgentTools.evaluate_emergency_risk() for deterministic regex trigger matching.
Emergency Branch (Risk Level: High / Med): Bypasses LLM text generation entirely. Calls PatientMemoryDB.create_escalation(), which extracts a point-in-time profile snapshot, writes an escalation payload to the escalations table, and logs the event to audit_logs.
Standard Branch (Risk Level: Low): Proceeds to Agent.chat(). Contexts inject system prompts and active memory XML tags, query the local Qwen 3.5 API, and handle function execution via the save_patient_record tool. The tool calls add_or_update_record() to UPSERT data into patient_records and log mutations to audit_logs.

 # Core Execution Flow
1. Edge PHI Scrubbing: Raw inputs pass through MedicalAgentTools.redact_phi() prior to LLM inference, replacing Malaysian MyKad IDs, phone numbers, emails, and names with anonymized tokens.
2. Deterministic Risk Assessment: Inputs bypass LLM processing and evaluate against regex triggers via "MedicalAgentTools.evaluate_emergency_risk()". High/Medium emergency conditions bypass text generation entirely and trigger an emergency protocol.
3. Context Injection & LLM Reasoning: For low-risk queries, active patient medical records are retrieved from patient_records and dynamically injected into the system context.

# Data Schema
1. patient_records Table
id: INTEGER PRIMARY KEY AUTOINCREMENT — Internal record identifier.
patient_id: TEXT NOT NULL — Unique identifier linking the medical record to a specific patient.
category: TEXT NOT NULL — Clinical category constrained to 'medication', 'allergy', 'symptom', 'chronic_condition', or 'chief_complaint'.
value: TEXT NOT NULL — Specific clinical attribute or entry (e.g., 'Panadol').
status: TEXT NOT NULL DEFAULT 'active' — Lifecycle status constrained to 'active', 'stopped', or 'resolved'.
provenance_pointer: TEXT — Unique interaction identifier (e.g., 'msg_20260903123000') linking the fact to its source message.
audio_transcript_id: TEXT — Nullable pointer reserved for linking voice-ingested source files.
reported_at / updated_at: DATETIME DEFAULT CURRENT_TIMESTAMP — Timestamps tracking record creation and state changes.
Constraints: UNIQUE(patient_id, category, value) — Composite key enabling SQLite ON CONFLICT DO UPDATE semantics.
2. escalations Table
id: INTEGER PRIMARY KEY AUTOINCREMENT — Unique escalation ticket identifier.
patient_id: TEXT NOT NULL — Links directly to patient_records.patient_id.
triggering_message: TEXT NOT NULL — PHI-scrubbed text that triggered the emergency flag.
triage_summary: TEXT NOT NULL — System-generated reason and rule match explaining the risk level.
profile_snapshot: TEXT NOT NULL — Serialized text snapshot of all active patient_records at the exact moment of triage.
provenance_pointer: TEXT — Interaction message ID tying the trigger to the execution turn.
acquisition_context: TEXT — Channel context string (e.g., 'terminal_session').
status: TEXT NOT NULL DEFAULT 'pending' — Clinical state constrained to 'pending', 'reviewed', or 'resolved'.
clinician_response: TEXT — Nullable field reserved for clinician review notes and instructions.
created_at: DATETIME DEFAULT CURRENT_TIMESTAMP — Record creation timestamp.
3. audit_logs Table
id: INTEGER PRIMARY KEY AUTOINCREMENT — Internal log entry identifier.
user_id: TEXT NOT NULL — Patient or system user associated with the operation.
event_type: TEXT NOT NULL — Audit event string (e.g., 'memory_mutation', 'escalation_sent').
metadata_json: TEXT NOT NULL — Structured, PHI-free JSON object capturing operational metadata.
timestamp: DATETIME DEFAULT CURRENT_TIMESTAMP — System execution timestamp.

# Ethics
TO BE DETERMINED

# First Principle Thinking
1. Determinism Over Prediction for Safety. Large language models are non-deterministic pattern matchers. Emergency safety checks must rely on deterministic code (MedicalAgentTools.evaluate_emergency_risk) that executes before invoking an LLM.

# Trade-offs
1. No Automated Medical Diagnosis: Nightingale acts purely as a triage and memory extraction assistant. Issuing formal medical diagnoses or drug prescriptions was cut to comply with healthcare legislation and eliminate clinical liability.
2. Local Functionality. It only works locally and unable to do real-world action such as calling emergency number.
