# Coding Standards for News Synthesizer

## General Coding Guidelines
- Follow PEP 8 for Python backend and TypeScript/Prettier standards for Next.js frontend
- Use meaningful variable names with consistent casing (snake_case for Python, camelCase for JS)
- Keep functions and methods concise (<30 lines when possible)
- Use type hints in Python and TypeScript interfaces
- Implement comprehensive error handling and logging

## Backend Standards (Python/FastAPI)
- Use asynchronous functions for all I/O operations (RSS fetching, API calls)
- Implement proper dependency injection with FastAPI dependencies
- Use SQLAlchemy with async ORM for database operations
- Follow REST API design principles with OpenAPI schema compliance
- Implement circuit breaker patterns for external service calls
- Log all operations with structured logging (structlog or built-in logging)

## Frontend Standards (Next.js/TypeScript)
- Use App Router for routing and server/client component separation
- Implement responsive design with Tailwind CSS
- Follow React best practices with hooks and custom hooks
- Use React Query (TanStack Query) for server state management
- Ensure TypeScript strict mode compliance
- Handle loading states and error boundaries for UX

## LLM and Model Integration Standards
- Validate all LLM inputs for safety and format compliance
- Implement token counting and usage monitoring
- Cache inference results to reduce redundant calls
- Handle model failures gracefully with fallbacks
- Monitor memory usage and performance metrics

## RSS and Data Processing Standards
- Validate RSS feed URLs and handle parsing errors
- Implement content deduplication and quality filtering
- Use streaming for large content processing
- Maintain data consistency across caching layers
- Sanitize all content to prevent XSS in generated outputs

## Database and Caching Standards
- Use SQLite for local development, PostgreSQL for production if needed
- Implement proper data migrations and schema versioning
- Index frequently queried fields for performance
- Use Redis or in-memory caching for fast data access
- Regular backup procedures for critical data

## Testing Standards
- Write unit tests for all critical functions (pytest minimum 80% coverage)
- Integration tests for API endpoints and component interactions
- Test LLM prompt engineering and output validation
- Performance tests for inference and RSS processing
- Accessibility tests for frontend components

## Security Standards
- Store sensitive configs in environment variables
- Implement input validation and sanitization
- Use HTTPS for all external communications
- Regular security scans with banditori and dependency checks
- Local model isolation to prevent data exfiltration

## Version Control
- Follow GitFlow branching model with semantic versioning
- Commit messages use conventional commits format:
  - `feat: add RSS sentiment analysis`
  - `fix: resolve LLM token limit handling`
  - `docs: update API mapping documentation`
- Pull requests require code review and tests passing

## Performance Standards
- LLM inference within 5s per article for metadata
- RSS processing under 10 minutes for 100 articles
- Frontend bundle size <5MB total
- Memory usage <8GB for typical workloads
- API response times <2s for non-LLM operations

## Accessibility Standards
- WCAG 2.1 Level AA compliance
- Screen reader compatibility for TTS-related features
- Keyboard navigation support throughout UI
- Sufficient color contrast and text sizing
- Alternative text for all non-text content

## Documentation Standards
- Update docs/ directory with each major change
- Include code examples and configuration samples
- Maintain CHANGELOG.md with release notes
- API documentation auto-generated from OpenAPI specs
- Inline code comments for complex logic

## Checklist
- [ ] Code formatted with black/isort for Python, prettier for TypeScript
- [ ] Static analysis passing (mypy, eslint)
- [ ] Tests passing with coverage targets
- [ ] Security scans clean
- [ ] Performance benchmarks met

## Ledger
| Standard Category | Last Reviewed | Status | Notes |
|-------------------|---------------|--------|-------|
| Coding Conventions | 2025-10-28 | Active | Async-first approach added |
| Security Practices | 2025-10-28 | Active | LLM isolation emphasized |
| Testing Requirements | 2025-10-28 | Active | Performance testing included |
