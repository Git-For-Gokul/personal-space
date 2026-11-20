# Enterprise Conversational Data Retrieval System

*Architecture overview and proof-of-concept proposal*

## 1. Problem statement

Enterprise applications expose large amounts of business data across multiple database tables, pages, and modules. Users often struggle to retrieve information because they need to understand the application's structure, field names, relationships, and search criteria.

Traditional keyword-based search requires extensive manual mapping of user phrases to backend fields. As the number of fields, modules, and possible user expressions increases, maintaining these mappings becomes difficult and less scalable.

The objective is to build an intelligent conversational interface that lets users retrieve enterprise information using natural language, without needing to know the underlying database schema or application structure.

## 2. Proposed solution

An NLP-driven conversational data retrieval architecture built on Java, semantic embeddings, and metadata-driven field resolution.

The system processes natural-language queries, identifies user intent, extracts relevant entities and conditions, and resolves business concepts to approved database metadata.

Instead of maintaining extensive phrase-to-field mappings, the solution uses semantic similarity to identify relevant business fields through their descriptions and embeddings.

For complex queries, an LLM-assisted structured extraction layer can convert natural language into validated JSON representations. A deterministic Java backend validates the requested fields, operators, and relationships before constructing any database query.

The architecture separates language understanding from database access, ensuring controlled, secure, and maintainable data retrieval.

## 3. Architecture

The pipeline has seven layers:

1. **Conversational interface:** accepts natural-language user queries.
2. **NLP / semantic understanding:** embeddings, cosine similarity, and intent detection.
3. **Entity and query extraction:** employee, date range, filters, and operators.
4. **Metadata / field resolver:** business field, then table, column, and join relationships.
5. **Java query builder and validator:** field validation, access control, and SQL generation.
6. **PostgreSQL:** executes approved queries and returns results.
7. **Conversational response:** presents retrieved information in natural language.

**Optional component: LLM-assisted extraction.** Used when complex queries require contextual understanding and multi-condition extraction. Its output is always validated by the Java layer before use.

## 4. Technology stack

| Component | Technology |
| --- | --- |
| Backend | Java 17, Spring Boot |
| NLP preprocessing | Tokenization, stemming, TF-IDF |
| Semantic understanding | Sentence embeddings |
| Similarity | Cosine similarity |
| Query extraction | Regex, rules, optional LLM |
| Metadata storage | PostgreSQL |
| Vector storage | PostgreSQL + pgvector |
| API integration | Java HttpClient / REST |
| Query execution | JDBC / SQL |

## 5. Key design decisions

1. **Reduced mapping complexity:** instead of maintaining hundreds of manually defined user phrases, the system uses canonical business descriptions and semantic embeddings.
2. **Separation of concerns:** NLP understands user intent, metadata resolves business fields, and Java controls database access.
3. **Hybrid NLP and LLM approach:** simple queries use embeddings and deterministic extraction; complex queries can use LLM-assisted structured extraction.
4. **Reusable metadata architecture:** new fields and database mappings can be introduced without changing the core NLP resolver.
5. **Controlled SQL generation:** the LLM never has unrestricted authority to generate and execute database queries. All queries are built from approved metadata, with field validation and access restrictions.

## 6. Current status and next steps

This is an architecture and proof-of-concept effort in progress. Not every component is production-implemented yet.

The next milestone is a proof of concept covering 8 to 10 fields, after which the design can be evaluated against measurable results.

## 7. Why we need pgvector

pgvector lets us store field embeddings and run similarity search inside the PostgreSQL database we already operate. That replaces hand-maintained phrase mappings with one governed, scalable lookup, and it needs no new server or platform.

### What the extension is

- pgvector is an open-source PostgreSQL extension that adds a vector column type and similarity operators (cosine distance, Euclidean distance, inner product).
- It is enabled per database with a single command, `CREATE EXTENSION vector`, and runs inside PostgreSQL itself.
- When the data grows, it offers approximate nearest-neighbour indexes (HNSW and IVFFlat) to keep searches fast.

### Why embeddings must be stored and searched

The system's core job is to map what a user says to the right business field. An embedding turns a piece of text into a list of numbers so that text with similar meaning produces similar numbers. For example, "vacation", "time off", and "annual leave" end up close together, even though they share no words.

To use this we need three things: generate a vector for each approved field once, store those vectors, and at every question find the stored vectors closest to the user's phrase. The second and third parts, storage and nearest-neighbour search, are exactly what pgvector provides inside PostgreSQL.

### Benefits

1. **Removes manual phrase mapping.** Fields are matched by meaning, so we do not maintain hundreds of user phrases per field.
2. **One governed data store.** Vectors sit next to the metadata catalog, linked by the same field key, and inherit the same backups, access control, and audit as the rest of the database. No second system has to be kept in sync.
3. **Similarity search combined with SQL filters.** In a single query we can limit candidates to the user's module, to fields their role may access, and to active fields, then rank by similarity. Unauthorized fields are never even candidates.
4. **Scales beyond the PoC.** The same design works from 10 fields to thousands of fields and example phrases, with indexes keeping response times low.
5. **Fits our existing stack.** It is used through standard JDBC and SQL from the Java backend, with no new client technology.
6. **Consistent updates.** A field and its vector are added, changed, or removed in one transaction, so the catalog and the vectors cannot drift apart.
7. **Data stays in our database.** Stored vectors never leave PostgreSQL, and no external vector service is involved.

### Alternatives considered

| Option | How it works | Advantages | Drawbacks |
| --- | --- | --- | --- |
| Keyword and TF-IDF matching only | Match words to fields using manual phrase lists | Nothing to install | Fails on synonyms and paraphrases, which is the problem being solved; the phrase lists grow without limit |
| In-memory similarity in Java | Store vectors in an ordinary column, load them at startup, compute cosine similarity in the application | Nothing to install; adequate for about 10 fields | Does not scale; every application instance holds a copy; cache refresh logic needed; cannot combine with SQL filters; rework needed later |
| Dedicated vector database (a separate product) | Run a separate vector search service | Built for very large vector workloads | New infrastructure to approve, secure, back up, and operate; data duplicated outside PostgreSQL; more than this use case needs |
| pgvector in PostgreSQL (recommended) | Vectors stored and searched inside the existing database | Single store, SQL filtering, indexing, no new service, simple rollback | One extension to approve and install on the database server |

### Scope note

For the first 8 to 10 field PoC alone, in-memory similarity in Java would technically work. pgvector is requested now so the PoC is built on the same storage design as the production path, avoiding a rebuild, and so we can validate filtered search, indexing, and response times on the real database.

## 8. Detailed step-by-step process

The process has three phases: one-time setup of the metadata and embeddings (Phase A), the runtime flow that handles every user question (Phase B), and the proof-of-concept rollout and evaluation (Phase C). Each step lists what is done, how, and what it produces. The running example is illustrative: *"Show me the leave taken by John in the last 3 months."*

### Phase A: Setup (done once, then updated whenever fields are added)

**Step A1. Select the scope of the first release.**

- Choose one module or page and 8 to 10 business fields, for example employee name, department, leave type, leave status, start date, end date, manager, and number of days.
- Pick fields that span at least two tables so the join logic is exercised.
- With business owners, mark each field as allowed or restricted for querying, and note which user roles may see it.
- Output: an approved field list with owners and access notes.

**Step A2. Build the metadata catalog.**

- Create catalog tables in PostgreSQL holding one row per business field.
- For each field record: a unique field key, a display name, the business description, the data type, the owning entity, and an active flag.
- Record the technical mapping: table name, column name, and the join path needed to reach the field from the main entity (for example leave table joined to employee table on employee ID).
- Record allowed operators per field (equals, contains, between, greater than, less than, in list) and the roles permitted to read it.
- Output: a catalog that is the single source of truth for what the system may query. New fields are added as data, not as code.

**Step A3. Write canonical descriptions and a few example phrases.**

- Write each description by meaning, in plain business language, for example: "The type of leave an employee has applied for, such as sick, casual, or earned leave."
- Mention the common synonyms inside the description, and add 3 to 5 example phrasings as a safety net, not hundreds of manual mappings.
- Have a business user review the descriptions, because their quality directly drives matching accuracy.
- Output: reviewed descriptions and example phrases per field.

**Step A4. Choose the embedding model and where it runs.**

- Choose a sentence-embedding model suited to short business text and compare a few candidates on a small sample of real phrases.
- Decide whether it runs inside our environment (for example a self-hosted model called from Java) or through an approved external service. If an external service were used, user questions would leave our network, so this must be approved separately.
- Fix the model name and version, because vectors from different models cannot be compared. Changing the model later means regenerating all vectors.
- Output: a documented model decision, including vector dimension.

**Step A5. Generate the field embeddings.**

- For each field, combine its display name, description, and example phrases into one text and pass it to the model.
- Normalize the vectors if the chosen similarity method expects it.
- Output: one vector per field, tagged with the model name and version.

**Step A6. Store the vectors in PostgreSQL with pgvector.**

- Enable the extension in the target database, add a vector column to the field table (or a linked table), and write the vectors against the field keys.
- Run a test similarity query to confirm results come back in the expected order.
- Create a vector index later, only if the field count grows enough to need one. For 10 fields, a plain scan is fast.
- Output: searchable field vectors inside the same database as the catalog.

**Step A7. Define the structured query format.**

- Define a JSON schema for the internal query: intent, target entity, list of filters (field key, operator, value), resolved date range, sort order, and result limit.
- This schema is the contract between the language-understanding layers and the Java backend. Both the rules path and the optional LLM path must produce exactly this shape.
- Output: a versioned JSON schema.

**Step A8. Define the security and operating rules.**

- Define role-based field access and row-level restrictions.
- Define a fixed allow-list of operators, a maximum number of conditions, a maximum result size, and a query time limit.
- Create a read-only database account that can only read the approved tables or views.
- Decide how rejected or unsafe requests are logged, and who reviews them.
- Output: written security rules and the restricted database account.

### Phase B: Runtime flow (executed for every user question)

**Step B1. Receive the question and identify the user.**

- The conversational interface sends the user's text and a session or user identifier to the Spring Boot REST API.
- Authenticate the user and load their role and permitted fields from the catalog, so access rules apply from the very first step.
- Apply input limits: maximum length, allowed characters, and a rate limit per user.
- Example: input is "Show me the leave taken by John in the last 3 months" from a user with the HR-viewer role.

**Step B2. Preprocess the text.**

- Keep the original text untouched and work on a normalized copy: lower-case, trimmed whitespace, standardized punctuation.
- Tokenize, remove stop words where helpful, and apply stemming or lemmatization so "taken", "taking", and "take" are treated alike.
- Protect literal values such as names, numbers, and dates from being altered, because they are used later as filter values.
- Output: normalized tokens plus the original text.

**Step B3. Detect the intent.**

- Decide what the user wants: a list of records, a count, a single value, a comparison, or a summary.
- Use a fast first pass with TF-IDF and keyword rules (for example "how many" suggests a count), then confirm with embedding similarity against a small set of intent descriptions.
- Example result: intent is "list records", entity is "leave".

**Step B4. Extract entities and conditions.**

- Use rules and regular expressions to pull out names, status words, numbers, and quoted values.
- Detect operators from wording: "more than 5 days" means greater than 5, "between March and May" means a range, "not approved" means not equal.
- Keep each extracted value together with the phrase it came from, so later steps can match phrase to field.
- Example: value "John" (a person name) and the time phrase "last 3 months".

**Step B5. Resolve date and time expressions.**

- Convert relative expressions such as "last 3 months", "this year", "since January", or "yesterday" into exact start and end dates, using the server's current date and a configured time zone.
- Define clearly what each expression means, for example whether "last 3 months" means the 3 months up to today or the previous 3 full calendar months, and apply the same rule every time.
- Always display the resolved dates in the answer so any misreading is visible to the user.
- Example: last 3 months resolves to an exact start date and today's date.

**Step B6. Resolve business fields with semantic similarity.**

- Take each unresolved phrase (for example "leave taken" and "John") and generate its embedding with the same model used in Step A5.
- Query PostgreSQL with pgvector for the closest field vectors, limited in the same query to active fields that the user's role may access.
- Return the top few candidates with their similarity scores.
- Example: "leave taken" ranks leave start date and leave status highest; "John" is matched to the employee name field because a person-name value was detected in Step B4.

**Step B7. Decide whether the match is trustworthy.**

- Accept a match only when its score is above a configured threshold and clearly ahead of the second-best candidate.
- If no field clears the threshold, or two fields are nearly tied, stop and ask the user a short clarifying question that lists the candidate fields, for example "Do you mean leave start date or leave approval date?"
- Never guess silently on a low-confidence match. A wrong silent guess gives the user wrong data with no warning.
- Output: a list of resolved field-and-value conditions, or a clarifying question.

**Step B8. Route complex questions to the optional LLM layer.**

- Apply a rule-based complexity check. Send a question to the LLM only when it has multiple conditions, and/or logic, negation, comparisons across fields, or when the resolver's confidence is low.
- Simple questions stay on the embedding and rules path, which is faster, cheaper, and fully deterministic.
- For complex questions, send the question, the list of allowed field keys with their descriptions, and the JSON schema from Step A7, and instruct the model to return only JSON that conforms to the schema.
- The LLM never sees table names, column names, credentials, or any data from the database, and it never writes SQL.
- Output: the same structured JSON that the rules path would produce.

**Step B9. Validate the structured query in Java.**

- Parse the JSON and validate it against the schema.
- Check that every field key exists in the catalog and is active, every operator is allowed for that field, every value matches the field's data type, and the user's role may access every field.
- Enforce limits on the number of conditions and the size of the date range.
- Reject anything that fails, whether it came from the rules path or the LLM path, and return a safe message to the user.
- Output: a validated, trusted query object.

**Step B10. Resolve joins from the catalog.**

- Use the join paths stored in the catalog to determine which tables are needed for the selected fields.
- Build the join order deterministically from the catalog, never from model output.
- Example: leave details and employee name require the leave table joined to the employee table.

**Step B11. Build the SQL safely.**

- Generate the SQL in Java from the validated query object and the catalog's table and column names.
- Pass every user-supplied value (names, dates, numbers) as a bound parameter in a prepared statement. Never join user text into the SQL string.
- Add the row-level restrictions for the user's role and a maximum row limit.
- Output: a parameterized SQL statement and its parameter list.

**Step B12. Execute the query.**

- Run it through JDBC using the read-only database account from Step A8.
- Set a statement timeout so an expensive query cannot slow the database.
- Handle empty results, timeouts, and errors with clear user-facing messages, and never expose raw database errors.

**Step B13. Generate the response.**

- Turn the result set into a short natural-language answer, with a table for lists.
- State the interpretation used: the fields matched, the filters, and the resolved date range, so the user can verify it.
- Where results were capped, say so, and offer follow-ups such as narrowing the date range or adding a filter.
- Example: "John had 3 leave records between the resolved start and end dates," followed by a table of leave type, dates, and status.

**Step B14. Log for audit and improvement.**

- Record the question, detected intent, matched fields with their scores, the structured query, whether the LLM path was used, and the outcome, subject to the organization's data-handling rules.
- Review clarification requests and rejected queries regularly. They show which field descriptions need improvement.
- Use user feedback (helpful or not helpful) to build the test set used in Phase C.

### Phase C: Proof-of-concept rollout and evaluation

**Step C1. Build the end-to-end slice.**

- Implement Steps B1 to B13 for the 8 to 10 selected fields, using the embeddings and rules path first, without the LLM.
- Use a development database with the pgvector extension enabled.
- Output: a working demo that answers questions on the selected module.

**Step C2. Create the test set.**

- Write 50 to 100 realistic questions covering every field, each with its expected structured query.
- Include paraphrases, synonyms, misspellings, multi-condition questions, ambiguous questions, and deliberately unsafe or out-of-scope requests.
- Have a business user contribute questions, because they phrase things differently from developers.

**Step C3. Measure.**

- Track field-resolution accuracy, filter and date extraction accuracy, the rate of clarification requests, end-to-end response time, and the share of unsafe requests correctly blocked.
- Record the results as the baseline.

**Step C4. Tune.**

- Adjust confidence thresholds, rewrite weak field descriptions, add example phrases where needed, and fix extraction rules, based on the failures found.
- Re-run the test set after every change so improvements are measured, not assumed.

**Step C5. Add the LLM-assisted layer.**

- Enable it for complex questions only, behind the same validator.
- Compare accuracy, response time, and cost against the rules path on the same test set, and keep it only where it adds clear value.
- Confirm the approved hosting and data-handling position for the model before any real user data is used.

**Step C6. Run the security review.**

- Test attempts to reach restricted fields, attempts to inject instructions or SQL through the question text, requests that would return too much data, and cross-role access.
- Confirm that the validator, parameterized SQL, read-only account, row limits, and timeouts block each one.

**Step C7. Pilot and decide.**

- Release to a small group of real users, collect feedback, and review logs weekly.
- Decide on expanding to more modules using the measured results, and update this document with the achieved numbers.

### Control points at a glance

- Language understanding never touches the database directly.
- Every query is built by Java code from validated structure and approved metadata.
- Ambiguous input leads to a clarifying question, not a guess.
- Access is limited by role, a read-only account, row limits, and timeouts.
- The LLM, where used, only produces a structured request that Java validates, and never writes or runs SQL.
