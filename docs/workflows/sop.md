# Standard Operating Procedures for News Synthesizer

## Overview
This document outlines the standard operating procedures for developing, maintaining, and deploying the News Synthesizer application. These procedures ensure consistent handling of RSS feeds, LLM inference, persona management, and system operations while maintaining data privacy and performance.

## Project Initialization
1. Verify hardware requirements (GPU, RAM) for llama.cpp
2. Clone project structure and initialize submodules
3. Set up virtual environment for Python backend
4. Download and place GGUF model file
5. Configure RSS feed sources and persona files
6. Initialize Git repository with departmental branch structure

## Development Workflow
### Daily Operations
1. Pull latest changes and update dependencies
2. Review open issues and prioritize tasks
3. Test local model inference and RSS fetching
4. Implement changes following branching strategy:
   - `main` → Stable production releases
   - `dev` → Development integration branch
   - `feature/{component}` → New features (rss-parsing, llm-inference, etc.)
   - `hotfix/{issue}` → Critical bug fixes

### Coding Standards
- Follow standards.md for code formatting and conventions
- Use type hints in Python (mypy compliance)
- Implement comprehensive error handling for RSS failures
- Validate LLM outputs against expected formats

## Model and Inference Procedures
### Model Loading
1. Verify GGUF file integrity before loading
2. Test inference with sample prompts
3. Monitor memory usage during initial inference
4. Establish baseline performance metrics

### RSS Processing Operations
1. Daily feed validation: Check feed URLs are accessible
2. Content deduplication: Use hashing and cache checks
3. Processing prioritization: Freshness, relevance, quality scores
4. Error handling: Circuit breaker for failed feed fetches

### Persona Management
1. Backup existing persona files before modification
2. Test persona changes with sample compositions
3. Validate YAML schema compliance
4. Update frontend integration if new parameters added

## Chat Interface Operations
1. Monitor chat request patterns for abuse detection
2. Log persona adjustments with timestamps
3. Backup chat session data for continuity
4. Performance: Participate in TTS generation

## TTS and Audio Procedures
1. Test audio output device compatibility
2. Monitor TTS service availability and fallback options
3. Configure voice settings for persona consistency
4. Handle audio queuing and playback errors gracefully

## Monitoring and Maintenance
### Daily Checks
- Confirm model loading and inference capability
- Test RSS feed connectivity and processing
- Verify TTS service and audio playback
- Check system resource usage (CPU, RAM, GPU)

### Weekly Maintenance
- Update dependencies and patch security vulnerabilities
- Review performance logs and optimize bottlenecks
- Backup model files and configuration data
- Test disaster recovery procedures

### Monthly Review
- Analyze RSS processing success rates
- Review inference quality and accuracy metrics
- Assess persona usage and effectiveness
- Plan infrastructure upgrades based on growth

## Incident Response
### RSS Feed Failures
1. Identify affected feeds and alternative sources
2. Implement temporary fallbacks if available
3. Notify users of processing delays
4. Document root cause and implement fixes

### Model Inference Issues
1. Switch to CPU-only inference if GPU fails
2. Restart services and test model loading
3. Check memory allocation and clear caches
4. Revert to backup model if current model corrupted

### System Overload
1. Implement rate limiting on RSS fetches
2. Queue requests during high load periods
3. Monitor resource usage with alerting
4. Scale model instances if possible

## Backup and Recovery
- Daily backups of processed articles database
- Weekly backups of model files and configurations
- Monthly checkpoints of complete system state
- Recovery procedures: Step-by-step restoration with validation tests

## Documentation Procedures
- Update relevant .md files after all changes
- Maintain checklist.md and ledger.md concurrently
- Document all operational changes and decisions
- Review documentation accuracy monthly

## Checklist
- [ ] SOP review completed monthly
- [ ] All procedures tested in staging environment
- [ ] Backup procedures verified
- [ ] Emergency contacts documented

## Ledger
| Procedure | Last Reviewed | Status | Notes |
|-----------|---------------|--------|-------|
| Daily Operations | 2025-10-28 | Active | GPU monitoring added |
| Model Management | 2025-10-28 | Active | GGUF integrity checks |
| Incident Response | 2025-10-28 | Planned | Templates for common issues |
