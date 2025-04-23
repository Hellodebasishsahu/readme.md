# Automated News Pipeline for Non-Redundant Topic Tracking  
**Purpose**: Design a fully automated pipeline to track and surface high-quality, non-redundant news articles for complex narratives (e.g., "Supreme Court vs Parliament" in India), ensuring each article covers a unique topic or event for LLM processing.

---

## Executive Summary

This report outlines a fully automated pipeline to collect, process, and select news articles for complex narratives, ensuring **non-redundant topics** (i.e., no two articles cover the same topic or event). The pipeline leverages **Google News** for article collection, **Newspaper3k** for text extraction, a **Gemini-like LLM** for query generation and topic extraction, and **Sentence Transformers** for event clustering. Key features include strict deduplication at the topic/event level, optional coverage tracking for narrative completeness, and one-per-cluster article selection to guarantee uniqueness. The system addresses concerns about oversimplified deduplication, event-level uniqueness, and comprehensive coverage, producing a flexible number of articles (e.g., 3–10) for LLM summarization. The pipeline is implemented in ~150–250 lines of Python, with optional enhancements like GDELT for precise event detection.

---

## Introduction

The goal is to create a fully automated pipeline that surfaces non-redundant news articles for complex, evolving narratives, such as "Supreme Court vs Parliament" in India. Each selected article must cover a **unique topic or event** (e.g., VP remarks, basic structure doctrine) to avoid redundancy in downstream LLM processing (e.g., summarization). Unlike prior iterations aiming for a fixed 5–7 links, the number of articles is flexible, determined by the number of distinct topics/events. The pipeline uses Google News, Newspaper3k, a Gemini-like LLM, and clustering tools, ensuring event-level uniqueness, narrative completeness, and automation.

### Key Requirements
- **Non-Redundant Topics**: Each article must represent a unique topic/event, avoiding duplicates (e.g., multiple articles on the same VP remark).
- **Event-Level Uniqueness**: Deduplication based on events, not just text similarity.
- **Narrative Completeness**: All major sub-topics (e.g., VP remarks, opposition reactions) should be represented, with optional coverage tracking.
- **Automation**: Fully automated with no manual intervention.
- **Flexible Output**: Number of articles depends on unique topics (e.g., 3–10), not a fixed 5–7.

### Alignment with User Feedback
- **Prior Questions Addressed**:
  - **GDELT’s Role**: Optional for clustering to enhance event-level deduplication; Sentence Transformers is sufficient for most cases.
  - **MMR vs. One-per-Cluster**: One-per-cluster selection ensures non-redundancy, preferred over Maximal Marginal Relevance (MMR) for simplicity.
  - **Coverage Tracking**: Optional but recommended to ensure all sub-topics are covered.
- **Clarification**: The focus is on preventing redundant topics, not limiting to 5–7 links. One article per unique topic/event is selected.

---

## Pipeline Architecture

The pipeline consists of nine steps, fully automated using open-source tools and APIs. Each step ensures non-redundancy and comprehensive coverage.

### 1. Narrative Seed
- **Description**: Define the narrative seed (e.g., "Supreme Court vs Parliament") or detect trends from Google News.
- **Implementation**:
  - **Manual Seed**: Store in a config file (`config.json`: `"seed": "Supreme Court vs Parliament"`).
  - **Automated Trend Detection**: Use `pygooglenews` to fetch trends:
    ```python
    from pygooglenews import GoogleNews
    gn = GoogleNews(country='IN')
    trends = gn.search('Supreme Court')
    seed = trends['entries'][0]['title']  # Top trend
    ```
- **Tools**: `pygooglenews` ([pygooglenews](https://github.com/kotartemiy/pygooglenews)).
- **Automation**: Fully automated via config or trend detection.
- **Output**: A single narrative seed (string).

### 2. Queries and Sub-Queries via Gemini-like LLM
- **Description**: Generate 5–10 diverse queries to capture sub-topics (e.g., VP remarks, legal doctrines).
- **Implementation**:
  - Prompt the LLM: "Generate 5–10 diverse search queries for 'Supreme Court vs Parliament,' covering sub-topics like official remarks, legal doctrines, opposition reactions, and scholar views."
  - Example output: ["VP comments on Supreme Court," "basic structure doctrine India," "opposition reactions to judicial overreach," "legal scholar views on Supreme Court vs Parliament," "parliament response to judicial rulings"].
  - API call:
    ```python
    from gemini_api import GeminiClient  # Hypothetical API
    client = GeminiClient(api_key="your_key")
    prompt = "Generate 5–10 diverse search queries for 'Supreme Court vs Parliament'..."
    queries = client.generate(prompt)
    ```
- **Tools**: Gemini-like LLM API (e.g., Google Gemini, OpenAI GPT).
- **Automation**: Fully automated via API.
- **Output**: List of 5–10 queries.

### 3. News Collection via Google News
- **Description**: Collect ~20–100 articles from Google News for each query.
- **Implementation**:
  - Use `pygooglenews` to fetch articles from the past week:
    ```python
    articles = []
    for query in queries:
        results = gn.search(query, when='7d')
        articles.extend(results['entries'])
    ```
  - Filter for relevance (keyword overlap with seed) and source diversity (one article per domain).
- **Tools**: `pygooglenews`.
- **Automation**: Fully automated with query looping and filtering.
- **Output**: Article metadata (title, URL, source, date).

### 4. Text Extraction via Newspaper3k
- **Description**: Extract full text from articles for topic extraction and clustering.
- **Implementation**:
  - Use Newspaper3k:
    ```python
    from newspaper import Article
    article_texts = []
    for article in articles:
        try:
            news = Article(article['link'])
            news.download()
            news.parse()
            article_texts.append({'url': article['link'], 'text': news.text, 'title': news.title})
        except Exception as e:
            print(f"Failed to parse {article['link']}: {e}")
    ```
  - Skip articles with errors (e.g., paywalls).
- **Tools**: Newspaper3k ([Newspaper3k](https://newspaper.readthedocs.io/en/latest/)).
- **Automation**: Fully automated with error handling.
- **Output**: List of dictionaries with URL, text, and title.

### 5. Unique Topic Extraction from Each News Text
- **Description**: Extract the primary topic/event from each article (e.g., "VP remarks").
- **Implementation**:
  - Use Gemini-like LLM:
    ```python
    topics = []
    for article in article_texts:
        prompt = f"Summarize this article in one sentence and identify its main topic (e.g., VP remarks, legal doctrine): {article['text'][:2000]}"
        response = client.generate(prompt)
        topics.append({'url': article['url'], 'topic': response['topic'], 'summary': response['summary'], 'text': article['text']})
    ```
  - Alternative: BERTopic for scalability:
    ```python
    from bertopic import BERTopic
    model = BERTopic()
    topic_labels, _ = model.fit_transform([a['text'] for a in article_texts])
    for i, article in enumerate(article_texts):
        topics.append({'url': article['url'], 'topic': model.get_topic(topic_labels[i]), 'text': article['text']})
    ```
  - **Non-Redundancy**: Normalize topics (e.g., merge "VP comments" and "VP remarks" using LLM or cosine similarity).
- **Tools**: Gemini-like LLM or BERTopic ([BERTopic](https://maartengr.github.io/BERTopic/)).
- **Automation**: Fully automated.
- **Output**: Articles with unique topic labels.

### 6. Clustering via Unique Topic
- **Description**: Cluster articles by topics/events to deduplicate those covering the same event.
- **Implementation**:
  - **Primary Approach**: Sentence Transformers with DBSCAN:
    ```python
    from sentence_transformers import SentenceTransformer
    from sklearn.cluster import DBSCAN
    model = SentenceTransformer('all-MiniLM-L6-v2')
    embeddings = model.encode([a['text'][:1000] for a in topics])
    clustering = DBSCAN(eps=0.5, min_samples=2).fit(embeddings)
    clusters = clustering.labels_  # -1 for noise, 0, 1, ... for clusters
    ```
  - **GDELT’s Role**:
    - **When to Use**: Enhances clustering with pre-extracted events for global or complex narratives.
    - **Implementation**: Fetch events with `gdeltPyR`:
      ```python
      from gdelt import gdelt
      g = gdelt()
      events = g.Search(date=['2025-04-15', '2025-04-22'], table='events', country=['IN'], keyword='Supreme Court')
      ```
    - Map articles to events by date, location, and keywords.
    - **Decision**: Use Sentence Transformers for simplicity. GDELT is optional if clusters show redundancy.
  - **Non-Redundancy**: Each cluster represents a unique topic/event.
- **Tools**: Sentence Transformers ([Sentence Transformers](https://www.sbert.net/)), optionally `gdeltPyR` ([gdeltPyR](https://pypi.org/project/gdelt/)).
- **Automation**: Fully automated.
- **Output**: Articles grouped into clusters.

### 7. Coverage Tracking (Optional)
- **Description**: Ensure each sub-topic has at least one cluster.
- **Implementation**:
  - **With Coverage Tracking**:
    ```python
    subtopic_clusters = {}
    for i, topic in enumerate(topics):
        subtopic = topic['topic']
        cluster = clusters[i]
        if cluster != -1:
            subtopic_clusters.setdefault(subtopic, set()).add(cluster)
    for subtopic, cluster_set in subtopic_clusters.items():
        if len(cluster_set) < 1:
            new_queries = client.generate(f"Generate additional queries for {subtopic} in 'Supreme Court vs Parliament'")
            # Re-run steps 3–6
    ```
  - **Without Coverage Tracking**: Assume LLM queries and clustering cover all sub-topics.
  - **Recommendation**: Include for completeness (lightweight, ~20 lines).
- **Tools**: None additional.
- **Automation**: Fully automated if enabled.
- **Output**: Coverage confirmation or new queries.

### 8. Article Selection (One-per-Cluster)
- **Description**: Select one article per cluster to ensure unique topics.
- **Implementation**:
  - Select highest-quality article (e.g., by text length):
    ```python
    selected = []
    for cluster_id in set(clusters) - {-1}:
        cluster_articles = [i for i, c in enumerate(clusters) if c == cluster_id]
        best_idx = max(cluster_articles, key=lambda i: len(topics[i]['text']))
        selected.append(best_idx)
    selected_articles = [topics[i] for i in selected]
    ```
  - **Why One-per-Cluster?**:
    - Ensures no redundant topics, aligning with your goal.
    - Simpler than MMR, which risks selecting multiple articles from similar clusters.
  - **MMR Alternative**: Balances relevance and diversity but may include redundant topics.
- **Tools**: None additional.
- **Automation**: Fully automated.
- **Output**: Articles (one per cluster, e.g., 3–10).

### 9. Processing Link
- **Description**: Summarize or interpret selected articles with the LLM.
- **Implementation**:
  - Prompt:
    ```python
    article_texts_selected = [a['text'] for a in selected_articles]
    prompt = f"Summarize these articles, each covering a unique topic: {article_texts_selected}"
    summary = client.generate(prompt)
    ```
- **Tools**: Gemini-like LLM API.
- **Automation**: Fully automated.
- **Output**: Summary highlighting unique developments.

---

## Feedback Loop
- **Mechanism**: If coverage tracking (step 7) detects missing sub-topics, generate new queries and re-run steps 3–6.
- **Implementation**: Python script with conditional checks.
- **Purpose**: Ensures comprehensive coverage of unique topics.

---
## Optional: Maximal Marginal Relevance (MMR) and Alternative Diversity Tools

### Overview
While the primary pipeline uses a **one-per-cluster** selection method to ensure each article covers a unique topic or event, **Maximal Marginal Relevance (MMR)** and similar diversity-aware tools offer an alternative approach for article selection. MMR balances **relevance** to the narrative seed (e.g., "Supreme Court vs Parliament") with **diversity** among selected articles, potentially improving the quality and representativeness of the output. This section explores MMR and comparable tools, their applicability to the pipeline, and trade-offs compared to one-per-cluster selection, addressing the goal of non-redundant topic coverage for LLM processing.

### Maximal Marginal Relevance (MMR)
MMR is a selection algorithm that iteratively chooses articles to maximize relevance to the query (narrative seed) while minimizing similarity to already-selected articles. It uses the following formula:

\[
\text{MMR} = \arg\max_{d_i \in D \setminus R} [\lambda \cdot \text{Sim}_1(d_i, q) - (1 - \lambda) \cdot \max_{d_j \in R} \text{Sim}_2(d_i, d_j)]
\]

- **\(d_i\)**: Candidate article.
- **\(q\)**: Query (narrative seed).
- **\(R\)**: Set of already-selected articles.
- **\(\text{Sim}_1\)**: Similarity between article and query (e.g., cosine similarity of embeddings).
- **\(\text{Sim}_2\)**: Similarity between articles.
- **\(\lambda\)**: Trade-off parameter (0 ≤ \(\lambda\) ≤ 1, typically 0.5 for balance).

#### Implementation
- **Steps**:
  1. Compute embeddings for the narrative seed and all articles using Sentence Transformers.
  2. Initialize an empty set of selected articles.
  3. Iteratively select the article with the highest MMR score until the desired number of articles is reached or candidates are exhausted.
  4. Adjust \(\lambda\) to prioritize relevance (\(\lambda > 0.5\)) or diversity (\(\lambda < 0.5\)).
- **Code Example**:
  ```python
  from sentence_transformers import SentenceTransformer
  from sklearn.metrics.pairwise import cosine_similarity
  import numpy as np

  model = SentenceTransformer('all-MiniLM-L6-v2')
  seed_embedding = model.encode([seed])[0]
  article_embeddings = model.encode([a['text'][:1000] for a in topics])
  selected = []
  lambda_ = 0.5

  while len(selected) < len(set(clusters) - {-1}) and len(selected) < len(topics):
      scores = []
      for i, emb in enumerate(article_embeddings):
          if i not in selected:
              relevance = cosine_similarity([emb], [seed_embedding])[0][0]
              diversity = min([1 - cosine_similarity([emb], [article_embeddings[j]])[0][0] for j in selected], default=1)
              score = lambda_ * relevance + (1 - lambda_) * diversity
              scores.append((score, i))
      if scores:
          _, best_idx = max(scores)
          selected.append(best_idx)

  selected_articles = [topics[i] for i in selected]

---
## Tools and Dependencies
- **`pygooglenews`**: News collection ([pygooglenews](https://github.com/kotartemiy/pygooglenews)).
- **Newspaper3k**: Text extraction ([Newspaper3k](https://newspaper.readthedocs.io/en/latest/)).
- **Sentence Transformers**: Clustering ([Sentence Transformers](https://www.sbert.net/)).
- **Gemini-like LLM**: Query generation, topic extraction, summarization.
- **Optional**:
  - `gdeltPyR`: Event clustering ([gdeltPyR](https://pypi.org/project/gdelt/)).
  - BERTopic: Topic extraction ([BERTopic](https://maartengr.github.io/BERTopic/)).
- **Code Estimate**: ~150–250 lines of Python.

---

## Addressing Requirements
- **Non-Redundant Topics**: Ensured by topic extraction, clustering, and one-per-cluster selection. Topic normalization prevents splitting similar topics.
- **Event-Level Uniqueness**: Clustering (Sentence Transformers or GDELT) groups articles by events.
- **Narrative Completeness**: Optional coverage tracking ensures all sub-topics are represented.
- **Automation**: Fully automated with Python scripts and APIs.
- **Flexible Output**: One article per unique topic/event (e.g., 3–10 articles).

---

## Limitations
- **Google News**: May miss niche sources; supplement with LLM search if needed.
- **Newspaper3k**: Fails on paywalled sites; skip problematic articles.
- **LLM Costs**: API calls may incur costs; BERTopic reduces reliance for topic extraction.
- **Clustering Quality**: Tune DBSCAN parameters to avoid noisy clusters; GDELT can refine event detection.
- **Topic Normalization**: Requires LLM or similarity checks to merge similar topics.

---

## Example Output
For "Supreme Court vs Parliament":
- **Queries**: ["VP comments on Supreme Court," "basic structure doctrine India," "opposition reactions to judicial overreach," "legal scholar views"]
- **Articles**: 50 from Google News.
- **Topics**: "VP remarks," "basic structure doctrine," "opposition reactions," "legal scholar views," "parliament response"
- **Clusters**: 5 clusters.
- **Selected Articles**: 5 articles, one per cluster.
- **Summary**: "VP criticized judicial overreach, while scholars debated the basic structure doctrine..."

---

## Conclusion
The pipeline delivers a fully automated solution for non-redundant news tracking, selecting one article per unique topic/event using Google News, Newspaper3k, a Gemini-like LLM, and Sentence Transformers. GDELT is optional for enhanced clustering, one-per-cluster selection ensures non-redundancy, and coverage tracking supports completeness. With ~150–250 lines of Python, the system meets the requirements for complex narratives like "Supreme Court vs Parliament" as of April 22, 2025.

---

## References
- [pygooglenews](https://github.com/kotartemiy/pygooglenews)
- [Newspaper3k](https://newspaper.readthedocs.io/en/latest/)
- [Sentence Transformers](https://www.sbert.net/)
- [BERTopic](https://maartengr.github.io/BERTopic/)
- [gdeltPyR](https://pypi.org/project/gdelt/)
- [GDELT Project](https://www.gdeltproject.org/)