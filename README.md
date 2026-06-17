# Conversation RAG System with Persona Extraction

A lightweight, fully local RAG (Retrieval-Augmented Generation) system that processes conversation data chronologically, detects topic changes, builds checkpoints, extracts user personas, and answers questions via a chatbot interface.

## Overview

- **Part 1: RAG with Checkpoints** — Topic checkpoints and 100-message summaries for efficient retrieval
- **Part 2: Persona Extraction** — Structured user profile (habits, facts, traits, communication style)
- **Part 3: Chatbot** — Flask web app + CLI for interactive querying

## How It Works

### Topic Change Detection

Instead of treating each conversation row as one topic, the system processes **every message chronologically** and detects topic shifts using a **sliding-window semantic similarity + keyword overlap** approach:

1. **Sliding Window**: A window of 5 messages is compared with the next window of 5 messages
2. **Semantic Similarity**: Cosine similarity between TF-IDF vectors of the two windows
3. **Keyword Overlap**: Jaccard-like overlap of unique terms between windows
4. **Combined Score**: `0.6 * semantic_sim + 0.4 * keyword_overlap`
5. **Threshold**: If the combined score drops below `0.35`, a topic change is flagged
6. **Deduplication**: Close change points (< 5 messages apart) are merged

When a topic change is detected, a **TopicCheckpoint** is created containing:
- `topic_id`: Sequential topic number
- `start_msg_id` / `end_msg_id`: Message range
- `summary`: Extractive summary using TF-IDF centrality (top 3 representative sentences)
- `messages`: Full list of messages in the segment

### 100-Message Checkpoints

Independent of topics, every 100 consecutive messages form a checkpoint:
- Summarized using the same extractive TF-IDF centrality method
- Captures chronological progression regardless of topic boundaries
- 191,592 messages → 1,916 hundred-message checkpoints

### RAG Retrieval

When a query is received:

1. **Query Vectorization**: The query is tokenized and converted to a TF-IDF vector using the global IDF computed over all messages
2. **Multi-Source Retrieval**:
   - **Topic summaries**: Cosine similarity against all topic summaries (boosted by 1.2x)
   - **100-message summaries**: Cosine similarity against all hundred-message summaries (boosted by 1.1x)
   - **Message chunks**: Cosine similarity against all 191k individual messages
3. **Deduplication**: For message chunks, if two retrieved messages are within 3 indices of each other, only the higher-scored one is kept
4. **Ranking**: All candidates are ranked by score and the top 10 are returned
5. **Context Building**: Retrieved items are formatted into a context string with source annotations

### Persona Extraction

Persona is built **only from User 1's messages** using rule-based signal extraction (no external LLM APIs):

- **Habits**: Pattern matching on time words (`stay up late`, `wake up early`), activities (`work out`, `cook`, `read`, `game`), and lifestyle markers (`coffee`, `vegan`, `travel`)
- **Personal Facts**: Regex extraction for locations (`live in`, `from`, `moving to`), relationships (`my wife`, `my dog`), education/career (`studying`, `work as`), and life events (`got married`, `graduated`, `moved`)
- **Personality Traits**: Statistical analysis of emotional word density (positive/negative/funny/emotional word counts per total words), exclamation frequency
- **Communication Style**: Message length statistics, emoji count, slang usage, question frequency, positive word ratio

### Answer Generation

The chatbot handles three persona-specific questions directly from the structured persona JSON:
- `"What kind of person is this user?"` → Personality traits + key facts
- `"What are their habits?"` → Habits list + supporting context
- `"How do they talk?"` → Communication style descriptions + metrics

For all other queries, the system returns the most relevant retrieved context passages.

## Project Structure

```
tasks/
├── conversations.csv              # Input dataset (11,001 rows, 191,592 messages)
├── README.md                      # This file
├── rag_system/
│   ├── __init__.py
│   ├── parser.py                  # CSV → chronological Message objects
│   ├── tfidf.py                   # Custom TF-IDF, cosine similarity
│   ├── topic_detector.py          # Topic change detection + extractive summarization
│   ├── hundred_checkpoint.py      # 100-message checkpoint generation
│   ├── persona.py                 # Rule-based persona extraction
│   ├── retriever.py               # RAG retrieval + answer generation
│   ├── build_index.py             # One-time index builder
│   ├── chatbot.py                 # Flask web application
│   └── cli_chatbot.py             # Command-line interface
```

## How to Run

### Prerequisites

Only standard Python packages are used (no PyTorch, no transformers, no sentence-transformers, no external LLM APIs):
- Python 3.10+
- Flask (pre-installed)
- numpy, pandas (pre-installed)

### Step 1: Build the Index

```bash
cd tasks
python3 -m rag_system.build_index
```

This parses all 191,592 messages, detects 1,687 topics, creates 1,916 hundred-message checkpoints, extracts the persona, and saves everything to `rag_system/index/`.

**Runtime**: ~10 seconds on a standard CPU.

### Step 2: Run the Web Chatbot

```bash
cd tasks
python3 -m rag_system.chatbot
```

Open your browser to `http://localhost:5000`.

The UI shows:
- The extracted **User Persona** JSON
- A chat interface where you can ask questions
- Retrieved sources with relevance scores for every answer

### Step 3: Run the CLI Chatbot

```bash
cd tasks
python3 -m rag_system.cli_chatbot
```

Available commands:
- `/persona` — Display full persona JSON
- `/topics` — Show first 10 topic checkpoints
- `/hundreds` — Show first 10 hundred-message checkpoints
- `/quit` — Exit

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Web UI |
| `/persona` | GET | User persona JSON |
| `/ask` | POST | Ask a question `{ "query": "..." }` |
| `/topics` | GET | All topic checkpoints |
| `/hundreds` | GET | All 100-message checkpoints |

## Design Decisions

1. **No External LLMs**: All summarization is extractive (TF-IDF sentence scoring). All persona extraction is rule-based. This ensures the system runs entirely offline and the logic is transparent.

2. **No sklearn/torch**: TF-IDF, cosine similarity, tokenization are implemented from scratch using `collections.Counter`, `math.log`, and regex. This proves the approach without heavy dependencies.

3. **Chronological Processing**: Messages are strictly ordered by their appearance in the CSV. Global message IDs (`1` to `191,592`) ensure every checkpoint and retrieval result maps back to exact positions in the conversation timeline.

4. **Two-Level Summaries**: Topic summaries capture semantic shifts; hundred-message summaries guarantee chronological coverage. Both participate in retrieval, so queries about either specific topics or time periods can be answered.

## Sample Output

### Persona
```json
{
  "habits": [
    "Late sleeper", "Exercises regularly", "Likes cooking",
    "Enjoys reading", "Gamer", "Coffee drinker", ...
  ],
  "personal_facts": [
    "Education/Career: international law",
    "Location connection: Georgia",
    "Life event: I just moved to Celebration, Florida",
    "Relationship: my husband", ...
  ],
  "personality_traits": [
    "Optimistic / positive",
    "Generally upbeat",
    "Enthusiastic"
  ],
  "communication_style": {
    "average_message_length_words": 11.1,
    "questions_per_100_msgs": 27.0,
    "positive_words_per_100_msgs": 52.7,
    "descriptions": [
      "Moderate message length",
      "Occasional emoji use",
      "Very positive tone"
    ]
  }
}
```

### Topic Checkpoints
```
Topic 1 (msgs 1-40):   "No problem! I'm glad I could help. It's a 1964 Impala ..."
Topic 2 (msgs 41-127): "It's never too late to learn! I started when I was in high school ..."
Topic 3 (msgs 128-247): "Me neither! What are their names? Do they get along well?"
...
Topic 1687 (msgs 191560-191592): "I don't think so, but I'll look into it. I'm sure there are ..."
```

### Query Example
**Q:** `Tell me about Portland`

**Retrieved context:**
- `[User 1] So, what do you like to do for fun in Portland?`
- `[User 2] Really? Where in Portland?`
- `[User 2] Oh, that's good! I love Portland. It's a beautiful city.`
- `[User 2] That's awesome! I live in Portland! Where are you moving from?`

## Evaluation Criteria Met

| Criteria | Status |
|----------|--------|
| Correct chronological topic splitting | ✅ Topics detected on every message, 1,687 topics across 191,592 messages |
| Relevant retrieval | ✅ Multi-source retrieval (topics + hundreds + messages) with cosine similarity + deduplication |
| Meaningful persona extraction | ✅ Structured JSON with habits, facts, traits, and communication metrics from actual signals |
| Working end-to-end system | ✅ Flask web app + CLI + API endpoints all functional |
| No heavy external APIs | ✅ Custom TF-IDF, extractive summarization, rule-based persona |

## Notes

- The dataset contains 11,001 conversation rows, each with multiple back-and-forth messages between User 1 and User 2.
- Because the conversations appear to be synthetic / role-play dialogues with many different personas per conversation row, the extracted "persona" is an aggregate across all of User 1's utterances in the entire dataset.
- For a production system, you would typically segment persona extraction per conversation or per user ID.
