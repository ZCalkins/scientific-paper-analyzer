# scientific-paper-analyzer
Multi-agent AI system for scientific paper summarization and cross-paper analysis using AWS Bedrock, RAG, and FAISS.# 🔬 Scientific Paper Analyzer

A multi-agent AI system that summarizes scientific papers, extracts key findings, and enables comparative analysis across multiple papers using Retrieval-Augmented Generation (RAG).

Built on AWS with Amazon Bedrock, Lambda, API Gateway, DynamoDB, S3, and FAISS.

## Architecture

The system uses a lead-agent/subagent pattern where an orchestrator decomposes user requests and delegates to three specialized agents:

- **Summarizer Agent** — generates structured summaries following a consistent template (title, one-sentence summary, research question, methodology, key findings, limitations, implications, glossary)
- **RAG Retriever Agent** — embeds user queries and performs similarity search across chunked paper text to answer specific questions grounded in source material
- **Comparator Agent** — retrieves relevant passages from multiple papers and synthesizes cross-paper analysis

The orchestrator runs agents in parallel and performs a final synthesis pass to produce a cohesive response.

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | Streamlit | Paper upload and query interface |
| API | API Gateway | REST endpoints for upload and query |
| Compute | AWS Lambda | Serverless execution for all agents |
| Orchestration | Custom Python (Bedrock Converse API) | Multi-agent coordination and parallel execution |
| LLM (primary) | Amazon Nova Pro | Orchestration and summarization |
| LLM (secondary) | Amazon Nova Lite | Retrieval and comparison (cost optimization) |
| Embeddings | Amazon Titan Text Embeddings | Vector generation for RAG pipeline |
| Vector Search | FAISS | Similarity search over paper chunks |
| Storage | S3 | Raw PDF and extracted text storage |
| Database | DynamoDB | Paper metadata and cached summaries |
| Text Extraction | pdfplumber | PDF to text conversion |

## Features

- **PDF Upload** — upload papers as PDFs; text is automatically extracted, chunked, embedded, and indexed
- **Structured Summaries** — consistent 8-section output format designed for scientific papers
- **Question Answering** — ask specific questions about any uploaded paper with answers grounded in the source text
- **Cross-Paper Comparison** — compare methodologies, findings, or limitations across multiple papers
- **Cost Optimization** — routes tasks to appropriate model tiers (Nova Pro for reasoning, Nova Lite for retrieval) and caches summaries to avoid redundant inference
- **Evaluation Framework** — rubric-based assessment of summary quality across accuracy, completeness, and clarity

## Project Structure

```
scientific-paper-analyzer/
├── README.md
├── architecture/
│   └── capstone_architecture.mermaid
├── lambdas/
│   ├── summarizer/             # Paper summarization agent
│   │   └── lambda_function.py
│   ├── pdf_processor/          # PDF text extraction and chunking
│   │   └── lambda_function.py
│   ├── orchestrator/           # Lead agent that coordinates subagents
│   │   └── lambda_function.py
│   └── rag_retriever/          # RAG similarity search and QA
│       └── lambda_function.py
├── streamlit/
│   └── app.py                  # Frontend application
├── prompts/
│   └── system_prompts.py       # All system prompts centralized
├── config.py                   # Endpoints, model IDs, table names
├── tests/
│   └── test_papers/            # Sample papers for testing
├── evaluation/
│   ├── eval_rubric.md          # Summary quality rubric
│   └── eval_results.md         # Evaluation scores and analysis
└── requirements.txt
```

## Setup

### Prerequisites

- AWS account with Bedrock model access enabled (Nova Pro, Nova Lite, Titan Embeddings)
- Python 3.11+
- AWS CLI configured

### Installation

```bash
git clone https://github.com/YOUR_USERNAME/scientific-paper-analyzer.git
cd scientific-paper-analyzer
pip install -r requirements.txt
```

### AWS Resources

1. **S3 Bucket** — create `scientific-paper-analyzer-{account-id}` with `uploads/`, `extracted/`, and `embeddings/` prefixes
2. **DynamoDB Table** — create `paper-analyzer` with partition key `paper_id` (String) and sort key `record_type` (String)
3. **Lambda Functions** — deploy each function in `lambdas/` with the shared IAM role (see `architecture/iam_policy.json`)
4. **API Gateway** — create REST API with `POST /upload` and `POST /query` routes pointing to the appropriate Lambdas

### Running the Frontend

```bash
cd streamlit
streamlit run app.py
```

## Design Decisions

**Why a multi-agent system instead of a single prompt?**
Scientific papers contain distinct information types (methodology, findings, terminology) that benefit from specialized extraction. Running focused subagents in parallel produces higher-quality output than a single monolithic prompt while reducing per-agent token costs.

**Why FAISS over OpenSearch Serverless?**
For a portfolio project with a small corpus (dozens of papers, not millions), FAISS bundled as a Lambda layer provides sub-100ms search latency at zero additional infrastructure cost. OpenSearch Serverless would be the right choice at production scale.

**Why two model tiers?**
The summarizer and orchestrator require strong reasoning (Nova Pro). The retriever and comparator primarily format pre-retrieved content, so a lighter model (Nova Lite) handles them at lower cost. This mirrors real-world cost optimization strategies used in production AI systems.

**Why cache summaries in DynamoDB?**
Summarizing the same paper twice wastes inference cost. A cached summary lookup in DynamoDB costs fractions of a cent versus several cents for a Bedrock invocation. For repeated queries about the same paper, this reduces latency from seconds to milliseconds.

## Evaluation

Summaries are evaluated against a rubric covering:

- **Accuracy** — are the key findings correctly represented?
- **Completeness** — are all 8 template sections meaningfully addressed?
- **Appropriate Omission** — does the summarizer note missing information rather than fabricating content?
- **Clarity** — is technical terminology explained without sacrificing precision?

Results for the test paper set are documented in `evaluation/eval_results.md`.

## Cost Analysis

| Operation | Estimated Cost |
|-----------|---------------|
| Single paper summary (Nova Pro, ~800 tokens out) | ~$0.006 |
| RAG query (Nova Lite, ~500 tokens out) | ~$0.001 |
| Embedding generation (Titan, ~2000 token paper) | ~$0.0002 |
| Cached summary lookup (DynamoDB) | ~$0.0000005 |

Costs are estimates based on current Bedrock pricing and will vary with input length.

## Roadmap

- [x] Single-agent summarizer with system prompt
- [x] Streamlit frontend with API Gateway integration
- [ ] PDF upload and text extraction pipeline
- [ ] RAG retrieval with FAISS vector search
- [ ] Multi-agent orchestration with parallel execution
- [ ] Cross-paper comparative analysis
- [ ] Evaluation framework with scored rubric
- [ ] Deployment to Streamlit Community Cloud

## License

MIT
