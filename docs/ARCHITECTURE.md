## Overview
The Python `lm-service` runs the llama-cpp-python library to serve LLM inference.
It connects to the Rust `lm-orchestrator` via gRPC. The Rust core is the central component responsible for managing:
- agent pipelines (currently RAG only)
- databases (Redis, MongoDB, Qdrant)
- tools

In the future, it will be decomposed into microservices.

## Cascade RAG
RAG is not a task that requires an iterative agentic loop. There is an algorithm for how humans process information, and the agent should follow the same approach.
Furthermore, this project focuses on local pipelines. Local quantized models have a known tendency to fall into infinite loops. The cascade pipeline addresses both of these issues.

The pipeline executes exactly two LLM calls:
1. Pre-retrieval: analyzes intent, identifies information gaps, may call one tool
2. Post-retrieval: evaluates retrieved context, generates final answer

## Memory
Message history is stored as BSON documents in MongoDB. Messages have a complex structure (tool calls, subagents in the future), so relational SQL-like databases are not suitable here.
Redis is used as a cache layer. Its necessity for a fully local pipeline is still under consideration.
MongoDB documents include a field for the agent's memory called `summary`. The summary is updated using a sliding window algorithm:
- let `n` be: the number of recent messages to keep in context
- let `m` be: the batch size of new messages to summarize

Initially, the summary is empty. It is updated when the following condition is met: `total_messages > n && (total_messages - n) % m == 0`.
These parameters can be configured via environment variables:
```env
HISTORY_MAX_MESSAGES=4
SUMMARY_INTERVAL=2
```

## Documents
Each document is parsed into a tree data structure based on heading hierarchy. When chunks are extracted, the full heading path is prepended to the section text as markdown headers. This ensures that every chunk retains its document context, so semantically related chunks remain discoverable even when separated by unrelated sections.

Currently only Markdown is supported (parsed via comrak). Chunks are bounded by token limits (`MAX_TOKENS = 254`): oversized sections are split at sentence or list-item boundaries, and slices flush once they reach `MIN_TOKENS = 128` (`crates/lm-orchestrator/src/vector/split_pipelines/md.rs`).

The example below counts words instead of tokens so it stays readable — the code path is identical, and the `architecture_example_chunks` test pins this exact output. Source document `gradients.md`:

```md
# Gradients

A gradient points in the direction of the steepest increase of a function.

## Stochastic gradient descent

Stochastic gradient descent estimates the gradient from a random subset of the
training data. The subset is called a batch, and its size trades noise for
throughput: smaller batches produce noisier steps but more updates per epoch,
while larger batches give smoother estimates at a proportionally higher cost.
Because every update depends on only a few examples, the method scales to
datasets that do not fit in memory. The noise is not only tolerated: it also
pushes parameters away from sharp minima, which often improves generalization.
Implementations typically shuffle the data once per epoch and decay the
learning rate as training progresses. Momentum and adaptive step sizes reduce
the sensitivity to that choice. The estimate is computed from a single batch
rather than the full dataset, so each step costs a fraction of a full gradient
evaluation.

The mini-batch estimate is biased in theory but effective in practice.
Averaging the gradient over more examples reduces its variance, yet the
computational cost grows linearly with the batch size. Hardware favors batches
that fill a cache line or a GPU wave, so the chosen size is often the largest
one that still fits the memory budget. Batch sizes also interact with
normalization layers and learning-rate schedules, so tuning them together is
usually necessary. A batch that is too small leaves the hardware idle and makes
the loss curve jump; one that is too large converges in fewer, more expensive
steps without improving the final loss. Practical guidance is to pick the
largest batch that trains stably, then adjust the learning rate and the
schedule to match.
```

The three chunks carry the document name as a root heading plus the section ancestry. The heading path is prepended before splitting, so it appears in the first slice only:

```md
# gradients.md

# Gradients

A gradient points in the direction of the steepest increase of a function.
```

```md
# gradients.md

# Gradients

## Stochastic gradient descent
Stochastic gradient descent estimates the gradient from a random subset of the
training data.
The subset is called a batch, and its size trades noise for
throughput: smaller batches produce noisier steps but more updates per epoch,
while larger batches give smoother estimates at a proportionally higher cost.
Because every update depends on only a few examples, the method scales to
datasets that do not fit in memory.
The noise is not only tolerated: it also
pushes parameters away from sharp minima, which often improves generalization.
Implementations typically shuffle the data once per epoch and decay the
learning rate as training progresses.
Momentum and adaptive step sizes reduce
the sensitivity to that choice.
The estimate is computed from a single batch
rather than the full dataset, so each step costs a fraction of a full gradient
evaluation.
The mini-batch estimate is biased in theory but effective in practice.
Averaging the gradient over more examples reduces its variance, yet the
computational cost grows linearly with the batch size.
Hardware favors batches
that fill a cache line or a GPU wave, so the chosen size is often the largest
one that still fits the memory budget.
Batch sizes also interact with
normalization layers and learning-rate schedules, so tuning them together is
usually necessary.
A batch that is too small leaves the hardware idle and makes
the loss curve jump; one that is too large converges in fewer, more expensive
steps without improving the final loss.
```

```md
Practical guidance is to pick the
largest batch that trains stably, then adjust the learning rate and the
schedule to match.
```

## Vector Search
The current cascade pipeline uses a JSON response field to invoke tools. At present, the model calls exactly one tool per query -- either sparse search or dense search (a hybrid approach is planned for the future).

TF-IDF vectors serve as the sparse representation, using English stemming and vocabulary pruning. MiniLM embeddings provide the dense vectors via ONNX Runtime with mean pooling and L2 normalization.
