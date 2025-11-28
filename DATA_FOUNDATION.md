Data storage & retrieval
- Object store + lakehouse for cheap, scalable storage reproducible tables.
- Time-series/analytics DB to store telemetry and drift analytics over time.
- Document/search store for full-text search with filters and scoring
- Vector DB for nearest-neighbour over embeddings at low latency
- Feature store for consistent features for training and serving
- Streaming real‑time event ingestion and delivery 
- Graph DB for multi-hop relational reasoning
- Cache to serve data in real time

Orchestration and operations
Workflow for multi‑step processes, & human‑in‑the‑loop
Pipelines for data processing, transformations & dependencies
Trust & safety helps content moderation, PII detection/redaction
Security & governance for control, policy, guardrail enforcement
Task distribution to parallelize processing, autoscale, retries, rate limits
Monitoring & evaluation to measure metrics, logs, traces, drift and quality
Multi-agent coordination for complex problems, collaboration, shared context

Quick readiness checklist
Data: Do you have clean, governed, and discoverable datasets with lineage?
Features: Are features defined once and served consistently across train/serve?
Observability: Can you measure latency, cost, drift, and quality per model?
Security: Are policies, secrets, and access enforced end‑to‑end?
Compliance: Is PII handled with detection, masking, and audit trails?
Resilience: Do workloads autoscale and degrade gracefully under load?
Integration: Are events, APIs, and batch jobs standardized and versioned?
Cost control: Do you track unit economics (per‑request cost) and set budgets?
Ownership: Are domains and SLAs clear, with on‑call and runbooks?
Rollouts: Can you A/B, canary, and rollback models safely?
