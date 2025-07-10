# SQL Workshop - AI Functions with Starburst

**Workshop Cluster Link:** [https://ai-workshop.enablement.starburstdata.net](https://ai-workshop.enablement.starburstdata.net)

---

## General Instructions

For all exercises, select the **catalog: **``

---

## Lab 1: Discovering SQL AI Functions

### Objective

Explore available AI functions in your Starburst SQL environment. Learn how to generate embeddings and apply various LLM functions.

### Instructions

Use role: `all_llm`

### SQL Script

```sql
-- List all AI functions
SHOW FUNCTIONS FROM starburst.ai;

-- List all configured language models
SELECT * FROM starburst.ai.language_models;

-- List all configured embedding models
SELECT * FROM starburst.ai.embedding_models;

-- Generate a vector embedding
SELECT ai.generate_embedding('Today is a fantastic day', 'nomic-embed-text');

-- Sentiment analysis examples
SELECT
    ai.analyze_sentiment('Today is a fantastic day', 'openai_small') AS user_quote,
    ai.analyze_sentiment('TSMC customers ordered fewer mobile chips...', 'openai_small') AS tech_report_summary;

-- Text classification
SELECT
    ai.classify('TSMC customers ordered fewer mobile chips...',
        array['life sciences', 'banking', 'tech', 'energy'],
        'openai_small') AS report_classification;

-- Grammar correction
SELECT
    ai.fix_grammar('That disaster effected so many lives', 'bedrock_claude35') AS incorrect_word,
    ai.fix_grammar('TSMC customers ordered fewer mobile chips...', 'openai_small') AS editorially_correct;

-- Masking sensitive data
SELECT ai.mask(
    'TSMC customers ordered fewer mobile chips... $25.8 billion... March 31.',
    array['financial data', 'dates'],
    'openai_small') AS masked_summary;
```

---

## Lab 2: Access Control for AI Models

### Objective

Test access restrictions to LLM models and AI functions based on SQL roles.

### Instructions

1. Use role: `limited_llm` (restricted access)
2. Then repeat using role: `all_llm`

### SQL Script

```sql
-- Translation examples
SELECT ai.translate('I like coffee', 'es', 'openai_small');
SELECT ai.translate('I like coffee', 'zh-TW', 'mistral');
SELECT ai.translate('I like coffee', 'fr', 'mistral');

-- Function access restriction test
SELECT ai.prompt('What is Starburst data?', 'openai_small');
```

---

## Lab 3: Loan Application Analysis with LLM

### Objective

Use an LLM to assess loan risk and make automated decisions based on application data.

### Instructions

Use role: `all_llm`

### SQL Script

```sql
-- View loan application data
SELECT * FROM iceberg.workshop.loan_approval;

-- Loan risk and decision
WITH loan_application_summary AS (
    SELECT
        text AS loan_reason,
        CONCAT('income: ', income, ', credit_score: ', credit_score, ', dti_ratio: ', dti_ratio, ', loan request: ', text) AS summary
    FROM iceberg.workshop.loan_approval
)
SELECT
    loan_reason,
    ai.classify(summary, array['high risk loan', 'moderate risk loan', 'low risk loan'], 'openai_small') AS loan_risk,
    ai.prompt(
        'You are a loan underwriter...Here is the loan application summary: ',
        summary,
        'openai_small') AS decision,
    summary AS loan_application
FROM loan_application_summary
LIMIT 10;
```

---

## Lab 4: Vector Search and Retrieval-Augmented Generation (RAG)

### Objective

Implement a full RAG architecture in SQL: generate embeddings, store them in an Iceberg table, perform similarity search, and enhance LLM answers with contextual data.

### Instructions

1. Find your username with: `SELECT current_user;`
2. Replace `<firstname>` with your username
3. Use catalog: `starburst`, schema: `ai`

### SQL Script (summarized steps)

```sql
-- Check current user
SELECT current_user;

-- Drop table if exists
DROP TABLE IF EXISTS iceberg.wks_users_schema.<firstname>_book_chapters;

-- Create table for FAA book chapters
CREATE TABLE iceberg.wks_users_schema.<firstname>_book_chapters (...);

-- Insert 10 mock rows (book chapters)
INSERT INTO iceberg.wks_users_schema.<firstname>_book_chapters (...)
VALUES (...);

-- Add embedding column
ALTER TABLE iceberg.wks_users_schema.<firstname>_book_chapters
ADD COLUMN chapter_intro_embeddings ARRAY(double);

-- Generate embeddings
UPDATE iceberg.wks_users_schema.<firstname>_book_chapters
SET chapter_intro_embeddings = ai.generate_embedding(chapter_intro, 'text-embedding');

-- Verify embeddings
SELECT chapter_intro_embeddings FROM iceberg.wks_users_schema.<firstname>_book_chapters;

-- Cosine similarity search
SELECT ... ORDER BY similarity_score DESC LIMIT 5;

-- Multiple similarity functions
SELECT chapter_intro, cosine_similarity(...), euclidean_distance(...), dot_product(...)
FROM iceberg.wks_users_schema.<firstname>_book_chapters
ORDER BY cs_score DESC;

-- RAG: LLM with JSON-based context
WITH vector_search AS (...),
     json_results AS (...)
SELECT ai.prompt(CONCAT('Which chapter should I read...', json_format(json_data)), 'openai_small') AS "LLM with RAG answer"
FROM json_results;

-- Cleanup
DROP TABLE IF EXISTS iceberg.wks_users_schema.<firstname>_book_chapters;
```

---

## End of Workshop

