# LLM Prompts for News Processing

## Overview
The system uses various prompts for different stages: metadata generation, RAG retrieval, synthesis, composition, and chat response.

## Metadata Generation Prompts
For each article, extract:
- Summary (2-3 sentences)
- Sentiment (positive/negative/neutral + score)
- Keywords (5-10)
- Topics

Prompt template: "Summarize this article in 2-3 sentences. Analyze sentiment. Extract keywords and topics."

## RAG Synthesis Prompts
Given a collection of articles and query:
"Given the following article metadata, retrieve and synthesize the most relevant information for creating a news segment on [query]."

## Composition Prompts
Using selected persona:
"You are [persona_name]. Compose a 300-word news segment based on this synthesized information. Follow these guidelines: [guidance]. Maintain this tone: [tone_words]."

## Chat Adjustment Prompts
Process user input to modify composition:
"Adjust the current persona parameters based on this user instruction: [user_message]. Update temperature, guidance, or tone as appropriate. Return the modified configuration."

## Prompt Engineering Standards
- Use clear instructions
- Include few-shot examples where needed
- Specify output format (JSON/XML)
- Limit token usage for efficiency

---

## Checklist
- [x] Prompts designed
- [ ] Prompt performance tested

## Ledger
| Prompt Type | Template | Performance Note |
|-------------|----------|------------------|
| Summary | "Summarize..." | Good accuracy |
| RAG | "Retrieve..." | Needs tuning |
| Compose
