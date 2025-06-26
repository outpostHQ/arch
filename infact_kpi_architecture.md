# Infact Analytics

## Architecture Overview for KPIs

### Query Flow
```mermaid
flowchart TB
  UserQ["'What programs have highest IRS for data science?'"]:::query
  UserQ --> Split["Break into KPI-specific sub‑goals"]:::answer

  subgraph KPIs["KPI Sub‑Queries"]
    Q1["List programs & course catalog feeds"]:::search
    Q2["Compute IRS: program courses vs. in‑demand skills"]:::compute
    Q3["Compute SMI/SGR: course‑to‑job skill mapping"]:::compute
    Q4["Compute ISD: skill frequency in employer postings"]:::compute
  end

  Split --> KPIs

  subgraph Fetch["Data Fetch"]
    D1["Program catalogs/websites"]:::source
    D2["Job postings (LinkedIn, Indeed, Glassdoor)"]:::source
    D3["Industry reports, syllabi APIs"]:::source
  end

  KPIs --> Fetch

  subgraph Pipeline["Processing Pipeline"]
    P1["Scrape & normalize course data (titles, descriptions, credits)"]:::process
    P2["Extract skills using NLP/ontology matching"]:::process
    P3["Retrieve industry job‑skill demand via NLP"]:::process
    P4["Align skills: course ↔ industry/job vocab"]:::process
  end

  Fetch --> Pipeline
  Pipeline --> CalcIRS["Calculate IRS = aligned courses / total courses"]:::process
  Pipeline --> CalcSMI["Calculate SMI = skill matches / total skills req'd"]:::process
  Pipeline --> CalcSGR["Calculate SGR = missing skills / total skills req'd"]:::process
  Pipeline --> CalcISD["Calculate ISD = avg skill mention frequency"]:::process

  subgraph KPICalc["KPI Computation"]
    CalcIRS
    CalcSMI
    CalcSGR
    CalcISD
  end

  KPICalc --> Validate["Validation & filtering (thresholds, confidence)"]:::validate
  Validate --> Synth["Synthesize results and rankings"]:::synthesis
  Synth --> Answer["Generate answer with visual scores & citations"]:::answer
  Answer --> UserResp["Complete response delivered"]:::answer

  classDef query fill:#ff8c42,stroke:#ff6b1a,color:#fff
  classDef primary fill:#ffd4b3,stroke:#ff6b1a
  classDef search fill:#ff8c42,stroke:#ff6b1a,color:#fff
  classDef compute fill:#42a5f5,stroke:#1e88e5,color:#fff
  classDef source fill:#3a4a5c,stroke:#2c3a47,color:#fff
  classDef process fill:#66bb6a,stroke:#388e3c,color:#fff
  classDef validate fill:#fb8c00,stroke:#ef6c00,color:#fff
  classDef synthesis fill:#8e24aa,stroke:#6a1b9a,color:#fff
  classDef answer fill:#546e7a,stroke:#37474f,color:#fff
```

1. User Query Breakdown
    - The system splits the main question into sub-questions targeting each KPI (IRS, SMI, SGR, ISD, etc.).
2. Data Fetching
    - Program catalogs: Scrape university/program sites for course lists and metadata.
    - Job postings: Collect skill requirements from platforms like LinkedIn, Indeed.
    - Industry reports/syllabi APIs: Include accreditation data and publicly available syllabi.
3. Processing Pipeline
    - Normalize scraped data (titles, descriptions, credit hours).
    - Use NLP and a skills ontology to extract and standardize skills mentioned.
    - From job postings and reports, build an industry-demand skill dataset.
    - Align and match skills between courses and industry/job taxonomies.
4. KPI Calculations
    - IRS: % of courses aligned with in-demand skills.
    - SMI: % of required job skills actually taught in courses.
    - SGR: % of job-required skills missing from course content.
    - ISD: Average frequency of skill mentions in job postings (demand intensity).
5. Validation & Synthesis
    - Filter by confidence thresholds (e.g. only keep IRS ≥ 80%).
    - Synthesize an answer: rank programs or courses, provide KPI breakdown, visualize.
6. Output
    - Present program/course recommendations with IRS, SMI, SGR, ISD scores.
    - Include supporting citations/links to scraped sources and job-skill data.

### Process Flow

```mermaid
flowchart TB
  UserQ["User KPI Query"] --> Split["Split into KPI Sub‑tasks"]
  
  Split --> A[Course & Job Data Sources]
  A --> B1["Scrape & Clean HTML/API data"]
  B1 --> C1["Normalized clean text"]
  
  C1 --> D1["NER (BERT-based)"]
  C1 --> D2["GPT‑4o-Mini (level capabilities)"]
  D1 & D2 --> E["Ensemble & Confidence Filter"]
  E --> F["Ontology Mapping (dictionary + embeddings)"]
  
  C1 --> G["Skill Mention Counting"]
  G --> H["SBERT Clustering / Synonym Grouping"]
  
  F --> KPI["Compute KPIs (IRS, SMI, SGR, ISD)"]
  H --> KPI
  
  KPI --> I["Validation (regex filters, confidence > 0.85, outlier detection)"]
  I --> J["Synthesis & Visualization"]
  J --> Output["Final Response"]

```

1. Data Ingestion & Normalization

| Stage                  | Model/Tool                                    | Why it fits                                                                  |
| ---------------------- | --------------------------------------------- | ---------------------------------------------------------------------------- |
| HTML/API scraping      | Lyra (Scraper from Outpost)                   | Lightweight, battle‑tested for arbitrarily‑structured web pages              |
| Raw text normalization | Lyra (Regex & Rule‑based)                     | Simple cleaning (remove HTML tags, boilerplate noise) is faster than ML here |

2. Skill Extraction (“Which skills does this course/job post mention?”)

| Stage                           | Model/Tool                                                | Why it fits                                                                                                              |
| ------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Entity / Skill phrase detection | Fine‑tuned NER (BERT‑based)                               | Pre‑trained on general English, fine‑tunable to recognize “Python”, “machine learning”, etc., with high precision/recall |
| Ontology mapping                | Dictionary lookup + embedding match (e.g. fastText)       | Catch synonyms (“PyTorch” → “python”) and map to canonical skill IDs                                                     |

Analogy:
> Think of Fine‑tuned NER (BERT‑based) as your “skill spotlight” that highlights every mention of a skill; the ontology matcher then says, “Ah, that spotlighted bit ‘PyTorch’ is really just a flavor of ‘Python.’”

3. Demand Extraction (“How often do employers ask for these skills?”)

| Stage                   | Model/Tool                                  | Why it fits                                                    |
| ----------------------- | ------------------------------------------- | -------------------------------------------------------------- |
| Job‑posting scraping    | Same as above                               | Keep your stack consistent                                     |
| Skill mention counting  | Count occurrences of extracted skill tokens | Simple frequency counts work as a first‑order proxy for demand |
| Advanced demand scoring | Sentence‑BERT embeddings + clustering       | Groups similar postings, smooths out noise across synonyms     |

Analogy:
> Fine‑tuned NER (BERT‑based) is like giving a waiter a list of detailed menu items and asking them to read every customer order, tagging every phrase that fits each menu description—even if the customer uses a creative or unusual name for the dish. The waiter doesn’t group similar items together; they simply spot and label every mention that matches your menu, no matter how it’s phrased.

4. KPI Computation & Matching

| KPI                         | Formula                                         | Matching mechanism                                                                |
| --------------------------- | ----------------------------------------------- | --------------------------------------------------------------------------------- |
| IRS (Industry Relevance)    | aligned\_courses / total\_courses               | Does “course\_skills” ∩ “industry\_top\_skills” ≥ threshold? Use set intersection |
| SMI (Skill Match Index)     | total\_skill\_matches / total\_skills\_required | For each target job role, compare extracted skill set vs. required\_skills list   |
| SGR (Skill Gap Ratio)       | missing\_skills / total\_skills\_required       | Complement of SMI                                                                 |
| ISD (Industry Skill Demand) | avg\_skill\_mention\_frequency across postings  | From demand extraction above                                                      |

How “Python” gets matched:

1. Extraction: Fine‑tuned NER (BERT‑based) has tagged “Python” in the course description.
2. Normalization: Ontology maps “Python 3” → skill_id=SKL_0001 (canonical).
3. Comparison:
    - If Python ∈ course_skills and Python ∈ industry_top_skills → it counts as an “aligned” course for IRS.
    - If Python is in required_skills for Data Engineer, it contributes to SMI.

5. Validation & Confidence

| Stage                    | Model/Tool                         | Role                                                                        |
| ------------------------ | ---------------------------------- | --------------------------------------------------------------------------- |
| False‑positive filtering | Rule‑based filters + regex checks  | E.g., ignore “python” when it’s in code samples snippet, not in description |
| Confidence scoring       | Embedding‑distance thresholds      | Only accept NER (BERT‑based) extractions whose tag confidence ≥ 0.85        |
| Outlier detection        | Isolation Forest or simple z‑score | Flag courses with extremely low/high IRS for manual review                  |

Analogy:
> Treat validation like a quality‑control line on an assembly belt—anything that falls below your confidence “height” or looks like an outlier gets kicked aside for manual QC.


