# Glossary

| Term | Definition |
|------|------------|
| **Agent** | LLM-based system that analyzes flags and provides recommendations |
| **BM25** | Best Matching 25—ranking function for text/token similarity |
| **Chunk** | Unit of data for comparison (sensor window, oil sample, event rollup) |
| **CMMS** | Computerized Maintenance Management System |
| **Distance weighting** | Closer neighbors have higher influence (weight = 1/distance) |
| **Event window** | Rolling time window (e.g., 14 days) of events for a single asset |
| **Failure-adjacent** | Chunks that preceded a failure event |
| **Feature vector** | Numeric representation of a chunk for distance calculation |
| **Flag** | Generated when new chunk scores above threshold |
| **Fleet baseline** | Statistical baseline across all assets of a model |
| **KNN** | K-nearest neighbors algorithm for similarity search |
| **pgvector** | PostgreSQL extension for vector similarity search |
| **Relevance score** | Weight based on component mapping |
| **SME** | Subject Matter Expert |
| **Synthetic memory** | SME-defined pattern without historical failure data |
| **Temporal score** | Label indicating proximity to failure (the training signal) |
| **Token** | Structured text representation of event (e.g., COMPONENT\|SUB\|TYPE) |
| **Trust score** | Weight based on source reliability |
| **tsvector** | PostgreSQL text search vector type |
| **TUI** | Terminal User Interface (Textual-based) |
