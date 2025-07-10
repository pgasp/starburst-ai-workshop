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
-- List of all AI functions
SHOW FUNCTIONS FROM starburst.ai;

-- List of all configured models
SELECT * FROM starburst.ai.language_models;

-- List of all embedding models
SELECT * FROM starburst.ai.embedding_models;

-- Generate vector embedding example (adjust model name based on availability)
SELECT ai.generate_embedding('Today is a fantastic day', 'nomic-embed-text');

-- Sample AI SQL functions
-- *** Delete the following before the demo:
-- Ensure outputs are tested beforehand as model behavior is non-deterministic
-- Cloud model results can be unpredictable
-- *** Delete above note before the demo

SELECT 
    ai.analyze_sentiment('Today is a fantastic day', 'openai_small') AS user_quote,
    ai.analyze_sentiment(
        'TSMC customers ordered fewer mobile chips in the first quarter than a year earlier, 
        but the company’s revenue nevertheless managed to top expectations.', 
        'openai_small'
    ) AS tech_report_summary;

SELECT
    ai.classify(
        'TSMC customers ordered fewer mobile chips in the first quarter than a year earlier, 
         but the company’s revenue nevertheless managed to top expectations.', 
        ARRAY['life sciences', 'banking', 'tech', 'energy'], 
        'openai_small'
    ) AS report_classification;

SELECT
    ai.fix_grammar('That disaster effected so many lives', 'bedrock_claude35') AS incorrect_word,
    ai.fix_grammar(
        'TSMC customers ordered fewer mobile chips in the first quarter than a year earlier, 
         but the company’s revenue nevertheless managed to top expectations.',
        'openai_small'
    ) AS editorially_correct;

SELECT ai.mask(
    'TSMC customers ordered fewer mobile chips in the first quarter than a year earlier, 
    but the company’s revenue nevertheless managed to top expectations.

    TSMC today posted sales of 839.25 billion New Taiwanese dollars, or $25.8 billion, 
    for the three months ended March 31. That’s up 41.6% from the same time a year earlier. 
    Analysts had expected slightly slower growth.',
    ARRAY['financial data', 'dates'],
    'openai_small'
) AS masked_summary;

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
-- Translation examples
SELECT ai.translate('I like coffee', 'es', 'openai_small');       -- Expected: Me gusta el café
SELECT ai.translate('I like coffee', 'zh-TW', 'mistral');         -- Expected: 我喜歡咖啡
SELECT ai.translate('I like coffee', 'fr', 'mistral');            -- Expected: J’aime le café

-- Unauthorized function access (should raise error)
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
-- Sample loan application data
SELECT * FROM iceberg.workshop.loan_approval;

-- LLM-based loan risk analysis and decision-making
WITH loan_application_summary AS (
    SELECT
        text AS loan_reason,
        CONCAT('income: ', income, ', credit_score: ', credit_score, 
               ', dti_ratio: ', dti_ratio, ', loan request: ', text) AS summary
    FROM iceberg.workshop.loan_approval
)
SELECT
    loan_reason,
    ai.classify(summary, ARRAY['high risk loan', 'moderate risk loan', 'low risk loan'], 'openai_small') AS loan_risk,
    ai.prompt(
        'You are a loan underwriter and provide decisions on loans based on applicant information. 
        You respond to a loan application ONLY with [Approved], [Rejected], or [More Info Needed], 
        followed by a brief 25-word explanation.

        Here is the loan application summary: ', summary, 'openai_small'
    ) AS decision,
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
-- Get your username
SELECT current_user;

-- Drop table if exists
DROP TABLE IF EXISTS iceberg.wks_users_schema.<firstname>_book_chapters;

-- Create table for mock FAA book data
CREATE TABLE iceberg.wks_users_schema.<firstname>_book_chapters (
  chapter_id BIGINT,
  book_title VARCHAR,
  book_id VARCHAR,
  publication_year BIGINT,
  chapter_number BIGINT,
  chapter_title VARCHAR,
  chapter_intro VARCHAR
);

-- Insert mock data (10 rows) – [truncated for brevity]

-- Add embedding column
ALTER TABLE iceberg.wks_users_schema.<firstname>_book_chapters 
ADD COLUMN chapter_intro_embeddings ARRAY(DOUBLE);

-- Generate embeddings
UPDATE iceberg.wks_users_schema.<firstname>_book_chapters
SET chapter_intro_embeddings = ai.generate_embedding(chapter_intro, 'text-embedding');

-- Verify embeddings
SELECT chapter_intro_embeddings FROM iceberg.wks_users_schema.<firstname>_book_chapters;

-- Cosine similarity search
SELECT DISTINCT 
    book_title,
    chapter_number,
    chapter_title,
    chapter_intro,
    cosine_similarity(
        ai.generate_embedding('Which chapter should I read to understand how to balance the weight of a Boeing 747?', 'text-embedding'), 
        chapter_intro_embeddings
    ) AS similarity_score
FROM iceberg.wks_users_schema.<firstname>_book_chapters
ORDER BY similarity_score DESC
LIMIT 5;

-- Compare similarity measures
SELECT 
    chapter_intro,
    cosine_similarity(
        ai.generate_embedding('Which chapter should I read to understand how to balance the weight of a Boeing 747?', 'text-embedding'), 
        chapter_intro_embeddings
    ) AS cs_score,
    euclidean_distance(
        ai.generate_embedding('Which chapter should I read to understand how to balance the weight of a Boeing 747?', 'text-embedding'), 
        chapter_intro_embeddings
    ) AS ed_score,
    dot_product(
        ai.generate_embedding('Which chapter should I read to understand how to balance the weight of a Boeing 747?', 'text-embedding'), 
        chapter_intro_embeddings
    ) AS dp_score
FROM iceberg.wks_users_schema.<firstname>_book_chapters
ORDER BY cs_score DESC;

-- RAG: Return top 5 chapters, then ask LLM for decision
WITH vector_search AS (
    SELECT DISTINCT 
        book_title,
        chapter_number,
        chapter_title,
        chapter_intro,
        cosine_similarity(
            ai.generate_embedding('Which chapter should I read to understand how to balance the weight of a Boeing 747?', 'text-embedding'), 
            chapter_intro_embeddings
        ) AS similarity_score
    FROM iceberg.wks_users_schema.<firstname>_book_chapters
    ORDER BY similarity_score DESC
    LIMIT 5
),

json_results AS (
    SELECT CAST(map_agg(chapter_number, json_object(
        KEY 'book title' VALUE book_title, 
        KEY 'chapter title' VALUE chapter_title, 
        KEY 'chapter intro' VALUE chapter_intro
    )) AS JSON) AS json_data
    FROM vector_search
)

SELECT 
    ai.prompt(CONCAT(
        'Which chapter should I read to understand how to balance the weight of a Boeing 747? Explain why. The list of chapters is provided in JSON: ', 
        json_format(json_data)
    ), 'openai_small') AS "LLM with RAG answer"
FROM json_results;

-- Drop table to clean up
DROP TABLE IF EXISTS iceberg.wks_users_schema.<firstname>_book_chapters;

```

---

## End of Workshop

