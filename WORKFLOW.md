# Workflow Explanation

## Overview

A multi-step agentic workflow built on n8n that automates negative review analysis for a D2C skincare brand. Takes a Google Sheets URL as input, fetches and filters reviews, runs two separate AI agents for classification and insights, waits for human approval, then writes a structured report to Google Docs.

---

## Workflow Structure

```
On Form Submission
       |
Get Sheet ID and GID
       |
  _____|___________
 |                 |
Get Reviews Data   Get Products Data
       |                 |
Filter Negative    Code in JavaScript1
Reviews            (drop rating col)
       |_________________|
               |
             Merge
               |
       Code in JavaScript
         (column selector)
               |
        _______|_______
       |               |
Loop Over Items    Loop Over Items2
       |               |
  Aggregate1        Aggregate
       |               |
   AI Agent         AI Agent1
  (Classify)        (Insights)
       |_______________|
               |
            Merge1
               |
         Result Parser
               |
      Code in JavaScript3
        (build report)
               |
             Wait
        (Human Approval)
               |
            Switch
        _______|_______
       |       |       |
    Approve  Show   Reject
       |     Here     |
  Create O/P  |     Form3
    Doc      Form2
       |
  Update O/P Doc
       |
     Form
  (Redirect to Doc)
```

---

## Node-by-Node Breakdown

### 1. On Form Submission
**Type:** Form Trigger
**What it does:** Entry point. User pastes their Google Sheets URL here.
**Why:** Replaces the manual step of downloading a CSV. Makes the workflow reusable for any sheet without touching the workflow itself.

---

### 2. Get Sheet ID and GID
**Type:** Code (JavaScript)
**What it does:** Extracts the Spreadsheet ID and tab GID from the URL using regex.
**Why deterministic:** Parsing a URL is a string operation with one correct answer.

```javascript
const url = $input.first().json['Reviews Sheet URL'];
const spreadsheetId = url.match(/\/d\/([a-zA-Z0-9-_]+)/)[1];
const gidMatch = url.match(/gid=(\d+)/);
const gid = gidMatch ? gidMatch[1] : '0';
return [{ json: { spreadsheetId, gid } }];
```

---

### 3. Get Reviews Data
**Type:** Google Sheets (Get Many Rows)
**What it does:** Fetches all rows from the reviews tab using the extracted spreadsheet ID and GID.

---

### 4. Get Products Data
**Type:** Google Sheets (Get Many Rows)
**What it does:** Fetches all rows from the product_info tab for metadata enrichment.

---

### 5. Filter Negative Reviews
**Type:** Code (JavaScript)
**What it does:** Filters to reviews with rating <= 1 and takes the top 50.
**Why deterministic:** The negative review threshold is a business rule, not a judgment call. AI should not decide what counts as negative.

```javascript
const filtered = items.filter(item => parseFloat(item.json['rating']) <= 1).slice(0, 50);
```

---

### 6. Code in JavaScript1
**Type:** Code (JavaScript)
**What it does:** Drops the `rating` column from the products sheet before merging to avoid column collision with the reviews rating column.
**Why deterministic:** Schema management is a code operation.

---

### 7. Merge
**Type:** Merge (Combine by Field)
**What it does:** Joins reviews and product data on `product_id` so every review has full product context.
**Why deterministic:** A join on a primary key is a lookup, not a reasoning task.

---

### 8. Code in JavaScript
**Type:** Code (JavaScript)
**What it does:** Keeps only the 8 columns needed for analysis — `rating`, `review_title`, `review_text`, `skin_type`, `product_id`, `product_name`, `brand_name`, `price_usd`. Drops everything else.
**Why deterministic:** Sending unnecessary columns to the LLM wastes tokens and increases the chance of rate limit errors.

---

### 9. Loop Over Items + Aggregate1 + AI Agent — Classification
**Type:** Loop, Aggregate, AI Agent (Groq / Llama 3.3 70B)
**What it does:** Loops through the filtered reviews in batches, aggregates each batch into a single item, and sends it to the LLM for classification.
**Why AI:** Classifying a review into the right issue bucket requires reading comprehension and judgment. A keyword filter would miss nuance and context.

**Buckets:**
- Delivery / Logistics
- Packaging
- Skin Reaction
- Efficacy / Results
- Texture / Consistency
- Smell / Fragrance
- Price / Value
- Authenticity
- Customer Service
- Other

**Output:** Structured JSON array — one object per review with `product_id`, `product_name`, `brand_name`, `review_title`, `bucket`.

---

### 10. Loop Over Items2 + Aggregate + AI Agent1 — Insights
**Type:** Loop, Aggregate, AI Agent (Groq / Llama 3.3 70B)
**What it does:** Runs all reviews through a second LLM agent that generates root cause analysis and recommendations.
**Why a separate agent:** Classification and analysis are different tasks. The classification agent needs to return strict JSON. The insights agent needs free-form prose. Mixing both in one prompt degrades output quality on both.

**Output covers:**
1. Top 3 critical issues and why they are happening
2. Most affected products and dominant complaint per product
3. 3 things the brand team should do based on the data

---

### 11. Merge1
**Type:** Merge (Append)
**What it does:** Combines the output of both AI agents — classification JSON and insights text — into a single 2-item payload.

---

### 12. Result Parser
**Type:** Code (JavaScript)
**What it does:** Parses the classification JSON string from AI Agent, extracts the insights text from AI Agent1, and returns a clean array of 26 items — 25 classified reviews plus 1 insights item.

```javascript
const reviews = JSON.parse(items[0].json.output);
const insights = items[1].json.output;
const result = reviews.map(item => ({ json: item }));
result.push({ json: { type: 'insights', insights } });
return result;
```

---

### 13. Code in JavaScript3
**Type:** Code (JavaScript)
**What it does:** Builds the full report string — bucket breakdown, insights, and review-level detail — and returns it as `{ report }` for use in both the Google Doc and the form completion page.

---

### 14. Wait
**Type:** Wait (On Form Submitted)
**What it does:** Pauses the workflow and sends an approval form URL to the brand manager via Gmail. The manager selects Approve, Show Here, or Reject.
**Why HITL:** The report influences real brand decisions. An AI-generated report should not bypass human validation before it reaches the team.

---

### 15. Switch
**Type:** Switch (Rules mode)
**What it does:** Routes to three branches based on the approval decision.
**Why deterministic:** Boolean routing on a form field value. No reasoning required.

| Decision | Branch |
|---|---|
| Approve | Create O/P Doc → Update O/P Doc → Form (redirect) |
| Show Here | Form2 (show report on page) |
| Reject | Form3 (rejection message) |

---

### 16. Create O/P Doc
**Type:** Google Docs (Create)
**What it does:** Creates a blank Google Doc with a timestamped title.

---

### 17. Update O/P Doc
**Type:** Google Docs (Update)
**What it does:** Writes the full report into the document.

---

### 18. Form / Form2 / Form3
**Type:** n8n Form

| Node | Mode | Purpose |
|---|---|---|
| Form | Redirect | Opens the generated Google Doc |
| Form2 | Completion Page | Shows the report inline on the page |
| Form3 | Completion Page | Shows rejection message |

---

## AI vs Deterministic Summary

| Step | Type | Reason |
|---|---|---|
| URL parsing | Deterministic | String operation |
| Data fetching | Deterministic | API call |
| Rating threshold filter | Deterministic | Business rule |
| Drop rating column | Deterministic | Schema management |
| Column selection | Deterministic | Token optimization |
| Join on product_id | Deterministic | Lookup operation |
| Batching | Deterministic | Rate limit management |
| Review classification | AI | Requires reading comprehension and judgment |
| Root cause insights | AI | Requires pattern recognition and reasoning |
| Approval routing | Deterministic | Switch on form value |
| Report building | Deterministic | String formatting |
| Document creation | Deterministic | API call |

---

## Agentic Practices Demonstrated

| Practice | Where |
|---|---|
| Role definition | Two agents with distinct roles — Classifier and Analyst |
| Structured outputs | Classification agent returns strict JSON schema |
| Tool use | Google Sheets, Google Docs, Gmail, Groq API |
| Branching / routing | Switch node on approval decision — 3 branches |
| Deterministic control | Filter, merge, column selection, batching, report building |
| Human-in-the-loop | Wait node with approval form before report generation |
| Fallback handling | Reject branch ends workflow cleanly with a message |
