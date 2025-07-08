# Travaux Pratiques SQL - Fonctions d’IA avec Starburst
Lien vers le cluster de l'atelier):[https://ai-workshop.enablement.starburstdata.net]

## TP1 : Découverte des fonctions d’IA SQL

### Objectif

Explorer les fonctions d’IA disponibles dans votre environnement SQL Starburst et apprendre à générer des embeddings ainsi qu'à appliquer différentes fonctions LLM.

### Instructions

- Utilisez un rôle : `all_llm` 

### Script SQL

```sql
-- Liste des fonctions d’IA disponibles
SHOW FUNCTIONS FROM starburst.ai;

-- Liste des modèles de langage configurés
SELECT * FROM starburst.ai.language_models;

-- Liste des modèles d'embedding configurés
SELECT * FROM starburst.ai.embedding_models;

-- Génération d’un vecteur d’embedding
SELECT ai.generate_embedding('Today is a fantastic day', 'openai_small');

-- Analyse de sentiment
SELECT
    ai.analyze_sentiment('Today is a fantastic day', 'openai_small') AS user_quote,
    ai.analyze_sentiment('TSMC customers ordered fewer mobile chips...', 'openai_small') AS tech_report_summary;

-- Classification de texte
SELECT
    ai.classify('TSMC customers ordered fewer mobile chips...',
                array['life sciences', 'banking', 'tech', 'energy'],
                'openai_small') AS report_classification;


-- Correction grammaticale
SELECT
    ai.fix_grammar('That disaster effected so many lives', 'bedrock_claude35') AS incorrect_word,
    ai.fix_grammar('TSMC customers ordered fewer mobile chips...', 'openai_small') AS editorially_correct;

-- Masquage d'information
SELECT ai.mask('TSMC customers ordered fewer mobile chips... $25.8 billion...',
               array['financial data', 'dates'],
               'openai_small') AS masked_summary;
```
---

## TP2 : Contrôle d’accès aux modèles d’IA

### Objectif

Tester les restrictions d’accès aux modèles LLM et aux fonctions AI selon les rôles SQL.

### Instructions

- Utilisez un rôle : `limited_llm` avec accès restreint à certaines fonctions et modèles
- Vérifiez les comportements attendus
- Repétez le test avec le rôle : `all_llm`

### Script SQL

```sql
-- Traduction
    SELECT ai.translate('I like coffee', 'es', 'openai_small');
-- Me gusta el café
    SELECT ai.translate('I like coffee', 'zh-TW', 'mistral');
-- 我喜歡咖啡
    SELECT ai.translate('I like coffee', 'fr', 'mistral');

-- Accès interdit à une fonction
    SELECT ai.prompt('What is Starburst data?', 'openai_small');
```

---

## TP3 : Analyse de demande de prêt avec LLM

### Objectif

Utiliser un modèle LLM pour évaluer le risque d’un prêt et prendre une décision automatisée à partir des données du dossier.

### Instructions

- Modifiez les noms de **catalogue** et **schéma** selon votre environnement
- Utilisez un rôle  `all_llm`
### Script SQL

```sql
-- Affichage des données de prêt
SELECT * FROM iceberg.workshop.loan_approval;

-- Analyse de risque et prise de décision
WITH loan_application_summary AS (
    SELECT
        text AS loan_reason,
        concat('income: ', income, ', credit_score: ', credit_score, ', dti_ratio: ', dti_ratio, ', loan request: ', text) AS summary
    FROM iceberg.workshop.loan_approval
)
SELECT
    loan_reason,
    ai.classify(summary, array['high risk loan', 'moderate risk loan', 'low risk loan'], 'openai_small') AS loan_risk,
    ai.prompt(
        'You are a loan underwriter and provide decisions on loans... summary: ',
        summary,
        'bedrock_claude35') AS decision,
    summary AS loan_application
FROM loan_application_summary
LIMIT 10;
```

---

## TP4 : Recherche vectorielle et génération augmentée (RAG)

### Objectif

Explorer une architecture RAG complète en SQL : génération d’embeddings, stokage des embeding comme colonne dans une table iceberg,  recherche de similarité, et réponse LLM augmentée par le contexte.

### Instructions

- Trouver votre nom d'utlisateur `select current_user`
- Rechercher et remplacer `<current_user>` par votre nom d'utilisateur
- Selectioner le catalog `starburst` ainsi que le schema `ai` afin de simplifier l'appel aux fonction ia

### Script SQL

```sql
SELECT current_user ;
drop  table if exists iceberg.wks_users_schema.<firstname>_book_chapters;

/*Create table for mock FAA book data*/
CREATE TABLE iceberg.wks_users_schema.<firstname>_book_chapters (
  chapter_id BIGINT,
  book_title VARCHAR,
  book_id VARCHAR,
  publication_year BIGINT,
  chapter_number BIGINT,
  chapter_title VARCHAR,
  chapter_intro VARCHAR
);

/*Populate table with mock FAA book data (10 rows)*/
INSERT INTO iceberg.wks_users_schema.<firstname>_book_chapters 
(chapter_id, book_title, book_id, publication_year, chapter_number, chapter_title, chapter_intro)
VALUES 
(1230, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 1, 'Weight and Balance Control', 'There are many factors in the safe and efficient operation of aircraft, including proper weight and balance control. The weight and balance system commonly employed among aircraft consists of three equally important elements: the weighing of the aircraft, the maintaining of the weight and balance records, and the proper loading of the aircraft. An inaccuracy in any one of these elements defeats the purpose of the system. The finalloading calculations are meaningless if either the aircraft has been improperly weighed or the records contain an error. Improper loading decreases the effic ency and performance of an aircraft from the standpoint of altitude, maneuverability, rate of climb, and speed. It may even be the cause of failure to complete the flight or, for that matter, failure to start the flight.Because of abnormal stresses placed upon the structure of an improperly loaded aircraft, or because of changed fly ng characteristics of the aircraft, loss of life and destruction of valuable equipment may result'),
(1231, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 2, 'Weight and Balance Theory', 'Weight and balance in aircraft is based on the law of the lever. This chapter discusses the application of the law of the lever and its applications relative to locating the balance point of a beam or lever on which various weights are located or shifted. The chapter also discusses the documentation pertaining to weight and balance that is furnished by the Federal Aviation Administration (FAA) and aircraft manufacturers.'),
(1232, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 3, 'Weighing the Aircraft and Determining the Empty Weight Center of Gravity', 'Chapter 2, Weight and Balance Theory, explained the theory of weight and balance and gave examples of the way the center of gravity (CG) could be found for a lever loaded with several weights. In this chapter, the practical aspects of weighing an airplane and locating its CG are discussed. Formulas are introduced that allow the CG location to be measured in inches from various datum locations and in percentage of the mean aerodynamic chord (MAC).'),
(1233, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 4, 'Light Sport Aircraft Weight and Balance Control', 'This chapter discusses the weight and balance procedures for light sport aircraft (LSA) that differ from conventional aircraft, specifically weight-shift control (WSC) aircraft (also called trikes), powered parachutes, and amateur-built LSA. [Figure 4-1]'),
(1234, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 5, 'Single-Engine Aircraft Weight and Balance Computations', 'Weight and balance data allows the pilot to determine the loaded weight of the aircraft and determine whether or not the loaded center of gravity (CG) is within the allowable range for the weight. See Figure 5-1 for an example of the data necessary for these calculations.'),
(1235, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 6, 'Multiengine Aircraft Weight and Balance Computations', 'Weight and balance computations for small multiengine airplanes are similar to those discussed for single-engine airplanes. See Figure 6-1 for an example of weight and balance data for a typical light twin-engine airplane.'),
(1236, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 7, 'Center of Gravity Change After a Repair or Alteration', 'The largest weight changes that occur during the lifetime of an aircraft are those caused by alterations and repairs. It is the responsibility of the FAA-certificatedmechanic or repairman doing the work to accurately document the weight change and record it in both the maintenance records and the Pilot’s Operating Handbook/Aircraft Flight Manual (POH/AFM)'),
(1237, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 8, 'Weight and Balance Control - Helicopter', 'Weight and balance considerations of a helicopter are similar to those of an airplane, except they are far more critical, and the center of gravity (CG) range is much more limited. [Figures 8-1 and 8-2] The engineers who design a helicopter determine the amount of cyclic control authority that is available, and establish both the longitudinal and lateral CG envelopes that allow the pilot to load the helicopter so there is sufficient cyclic control for all flight conditions'),
(1238, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 9, 'Weight and Balance Control - Commuter Category and Large Aircraft', 'This chapter discusses general guidelines and procedures for weighing large fixed-wingaircraft exceeding a takeoff weight of 12,500 pounds. Several examples of center of gravity (CG) determination for various operational aspects of these aircraft are also included. Persons seeking approval for a weight and balance control program for aircraft operated under Title 14 of the Code of Federal Regulations (14 CFR) part 91, subpart K, 121, 125, or 135 should consult with the Flight Standards District Office(FSDO) or CertificateManagement Office (CMO) that has jurisdiction in their area. Additional information on weight and balance for large aircraft can be found in Federal Aviation Administration (FAA) Advisory Circular (AC) 120-27, Aircraft Weight and Balance Control, FAA Type Certificate Data Sheets (TCDS), and the aircraft flight and maintenance manuals for specific aircraf'),
(1239, 'Aircraft Weight and Balance Handbook', 'FAA-H-8083-1B', 2016, 10, 'Use of Computer for Weight and Balance Computations', 'Almost all weight and balance problems involve only simple math. This allows slide rules and hand-held electronic calculators to relieve much of the tedium involved with these problems. This chapter compares the methods of determining the center of gravity (CG) of an airplane while it is being weighed. First, it shows how to determine the CG using a simple electronic calculator, then solves the same problem using an E6-B flightcomputer. Finally, it shows how to solve it using a dedicated electronic flight computer');

/*Create Embedding Column*/
ALTER TABLE iceberg.wks_users_schema.<firstname>_book_chapters ADD COLUMN chapter_intro_embeddings ARRAY(double);

/*Embed mock FAA book chapter introduction*/
UPDATE iceberg.wks_users_schema.<firstname>_book_chapters
SET chapter_intro_embeddings = ai.generate_embedding(chapter_intro, 'text-embedding');

/*Verify FAA book chapter embeddings created*/
SELECT chapter_intro_embeddings FROM iceberg.wks_users_schema.<firstname>_book_chapters;

/*Cosine similarity on mock FAA book chapters table*/
SELECT distinct 
    book_title,
    chapter_number,
    chapter_title,
    chapter_intro,
    cosine_similarity(
            ai.generate_embedding('Which chapter should I read to understand how to balance the weight of a Boeing 747?', 'text-embedding'), 
            chapter_intro_embeddings
            ) AS similarity_score
FROM
    iceberg.wks_users_schema.<firstname>_book_chapters
ORDER BY similarity_score DESC
LIMIT 5;


/*Similarity Search Functions*/
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
FROM
    iceberg.wks_users_schema.<firstname>_book_chapters
ORDER BY cs_score DESC;




/*RAG: Return top 5 similar documents, then ask LLM to select the best chapter*/
--CTE to return top 5 hitsw
WITH vector_search AS(
    SELECT distinct 
        book_title,
        chapter_number,
        chapter_title,
        chapter_intro,
        cosine_similarity(
            ai.generate_embedding('Which chapter should I read to understand how to balance the weight of a Boeing 747?','text-embedding'), 
            chapter_intro_embeddings) AS similarity_score
    FROM iceberg.wks_users_schema.<firstname>_book_chapters
    ORDER BY similarity_score DESC
    LIMIT 5
),

--CTE to cast results into JSON
json_results AS (
SELECT CAST(map_agg(chapter_number, json_object(
    key 'book title' VALUE book_title, 
    key 'chapter title' VALUE chapter_title, 
    key 'chapter intro' VALUE chapter_intro))AS JSON) AS json_data
FROM
    vector_search
)

--Augmented / Modified prompt to LLM w/ JSON results
SELECT 
    ai.prompt(concat(
        'Which chapter should I read to understand how to balance the weight of a Boeing 747? Explain why. The list of chapters is provided in JSON:', 
        json_format(json_data)), 'openai_small') as "LLM with RAG answer"
FROM json_results;

drop  table if exists iceberg.wks_users_schema.<firstname>_book_chapters;

```

---


