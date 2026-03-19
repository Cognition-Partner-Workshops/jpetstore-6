# Performance Testing Workshop with JPetStore-6 — Integrating Devin Data Analyst (DANA)

## 1. Overview

### Session Objectives

This workshop provides a hands-on, end-to-end performance testing experience using the JPetStore-6 application as the system under test. Participants will walk through the complete performance testing lifecycle — from test data generation and JMeter script creation through test execution, results analysis, and executive reporting — while learning how the **Devin Data Analyst (DANA)** agent can accelerate and enhance each phase.

### Target Audience

Performance testing professionals, QA engineers, and test architects who have working knowledge of:

- Apache JMeter (or comparable load testing tools)
- HTTP protocol fundamentals and web application architecture
- Basic statistics (percentiles, standard deviation, throughput)
- SQL and relational database concepts

### Application Under Test: JPetStore-6

JPetStore-6 is a Spring-based e-commerce pet store demo application built with:

| Component         | Technology                                      |
|-------------------|-------------------------------------------------|
| Web Framework     | Stripes MVC                                     |
| Business Layer    | Spring 5 (dependency injection, transactions)   |
| Persistence       | MyBatis 3 (mapper interfaces + XML SQL)         |
| Database          | HSQLDB (embedded, in-memory)                    |
| Build System      | Maven (with Maven Wrapper)                      |
| Deployment        | WAR on Tomcat 9 (Docker support included)       |

**Key application capabilities:**

- **Catalog browsing** — categories, products, items, search
- **User management** — registration, authentication, profile editing
- **Cart operations** — add items, update quantities, remove items
- **Order processing** — checkout, payment, shipping, order history

**Database schema** (14 tables): `supplier`, `signon`, `account`, `profile`, `bannerdata`, `orders`, `orderstatus`, `lineitem`, `category`, `product`, `item`, `inventory`, `sequence`

**Default dataset**: 7 users, 5 categories, 16 products, 28 items (each with 10,000 inventory units), 2 suppliers

> **Reference files:**
> - DB configuration: `src/main/webapp/WEB-INF/applicationContext.xml`
> - Schema definition: `src/main/resources/database/jpetstore-hsqldb-schema.sql`
> - Seed data: `src/main/resources/database/jpetstore-hsqldb-dataload.sql`

---

## 2. Session Agenda with DANA Integration Points

| Time (approx.) | Phase | Activity | DANA Integration |
|---|---|---|---|
| 0:00 – 0:30 | **Phase 1** | Application Walkthrough & Test Data Generation | Data analysis, realistic data generation |
| 0:30 – 1:15 | **Phase 2** | JMeter Test Script Creation | Endpoint coverage analysis, think-time recommendations |
| 1:15 – 1:45 | **Phase 3** | Test Execution & Monitoring | Real-time anomaly detection, correlation analysis |
| 1:45 – 2:30 | **Phase 4** | Test Run Analysis | **PRIMARY DANA SHOWCASE** — deep result analysis |
| 2:30 – 3:00 | **Phase 5** | Performance Test Summary Report | **STRONG DANA SHOWCASE** — automated report generation |

> **Legend:** Phases 4 and 5 are the strongest DANA integration points and should receive the most demo time.

---

## 3. Phase 1: Application Walkthrough & Test Data Generation

### 3.1 What the Presenter Does

1. **Walk through JPetStore architecture:**
   - Show the three-tier MVC structure (Stripes ActionBeans → Spring Services → MyBatis Mappers)
   - Open `applicationContext.xml` to explain the embedded HSQLDB DataSource, transaction manager, and MyBatis `SqlSessionFactory` configuration
   - Review `jpetstore-hsqldb-schema.sql` to illustrate the entity-relationship model:
     - `category` → `product` → `item` → `inventory` (catalog hierarchy)
     - `account` → `profile` + `signon` (user structure)
     - `orders` → `orderstatus` + `lineitem` (order structure)
     - `sequence` (ID generation)

2. **Identify key endpoints to test:**

   | Endpoint | HTTP Method | Description |
   |---|---|---|
   | `/catalog` | GET | Homepage / main catalog view |
   | `/catalog/viewCategory` | GET | Browse items by category (FISH, DOGS, CATS, REPTILES, BIRDS) |
   | `/catalog/viewProduct` | GET | View products within a category |
   | `/catalog/viewItem` | GET | View a specific item's details and price |
   | `/cart/addItemToCart` | GET/POST | Add an item to the shopping cart |
   | `/cart/viewCart` | GET | View current cart contents |
   | `/orders/newOrder` | POST | Submit a new order (checkout flow) |
   | `/account/signon` | POST | User authentication |
   | `/account/newAccount` | POST | User registration |

3. **Discuss test data challenges:**
   - Default dataset has only 7 users and 28 items — insufficient for realistic load testing
   - Need hundreds or thousands of test users with varied profiles
   - Need realistic order history to test reporting and read-heavy scenarios
   - Data must respect foreign key constraints and referential integrity

### 3.2 DANA Integration Point: Intelligent Test Data Generation

**Where DANA adds value:**

- **Schema and data distribution analysis** — Ask DANA to analyze the existing `jpetstore-hsqldb-schema.sql` and `jpetstore-hsqldb-dataload.sql` to map out all table relationships, column constraints, and current data distributions. DANA can identify that the current dataset is heavily skewed (e.g., all users share the same address, all inventory counts are identical at 10,000).

- **Realistic data generation** — DANA can generate a production-like SQL data file (targeting ~5 MB) with:
  - Hundreds of unique user accounts with varied addresses, email domains, and profile preferences
  - A realistic category preference distribution (e.g., 30% DOGS, 25% CATS, 20% FISH, 15% BIRDS, 10% REPTILES)
  - Order history with a **95/5 pass/fail ratio** to simulate real-world transaction patterns
  - Varied inventory levels across items to create realistic stock-out scenarios

- **Data dependency mapping** — DANA can produce a dependency graph showing the required insertion order:
  1. `supplier` → 2. `category` → 3. `product` → 4. `item` → 5. `inventory`
  6. `signon` → 7. `account` → 8. `profile`
  9. `sequence` → 10. `orders` → 11. `orderstatus` + `lineitem`

- **Volume and distribution recommendations** — Based on typical e-commerce patterns, DANA can suggest:
  - User-to-order ratio (e.g., 60% of users have placed at least one order)
  - Cart abandonment simulation (orders started but not completed)
  - Seasonal distribution patterns in order dates
  - Price distribution across item categories

**Suggested talking points:**

> "Instead of manually crafting SQL INSERT statements or writing custom data generation scripts, we can describe our data requirements to DANA in plain English and get a production-realistic dataset in minutes."

> "DANA understands relational constraints — it won't generate an order referencing a non-existent user or an item from a category that doesn't exist."

---

## 4. Phase 2: JMeter Test Script Creation

### 4.1 What the Presenter Does

1. **Create a JMeter Test Plan** with the following components:
   - **Thread Groups** — configure virtual user counts, ramp-up periods, and loop counts
   - **HTTP Request Samplers** — one for each endpoint identified in Phase 1
   - **HTTP Cookie Manager** — maintain session state across requests (critical for cart and order flows)
   - **CSV Data Set Config** — parameterize user credentials and item IDs from test data files
   - **Response Assertions** — validate HTTP status codes, response content patterns
   - **Listeners** — Summary Report, Aggregate Report, View Results Tree (for debugging)
   - **Timers** — Constant Timer or Gaussian Random Timer for think time simulation
   - **Transaction Controllers** — group related requests into logical transactions (e.g., "Browse and Add to Cart", "Checkout Flow")

2. **Build user journey scripts:**
   - **Browse-only journey**: Catalog → Category → Product → Item (read-heavy)
   - **Purchase journey**: Sign On → Browse → Add to Cart → View Cart → Checkout → Confirm Order
   - **Account creation journey**: New Account → Edit Profile → Browse

3. **Discuss parameterization and correlation:**
   - Extract dynamic values (session IDs, CSRF tokens) using Regular Expression Extractor or CSS/JQuery Extractor
   - Parameterize `categoryId`, `productId`, `itemId`, and user credentials from CSV files
   - Handle the multi-step order flow (`shippingAddressRequired` flag, `confirmed` flag)

### 4.2 DANA Integration Point: Script Design Assistance

**Where DANA adds value:**

- **User journey identification from logs or HAR files** — Provide DANA with application access logs or HAR (HTTP Archive) files captured during manual testing. DANA can identify:
  - The most frequently accessed endpoints
  - Common navigation paths through the application
  - The ratio of read operations (catalog browsing) vs. write operations (orders, account creation)
  - Critical business transactions that must be included in the test plan

- **Think time and pacing recommendations** — DANA can analyze session data to recommend:
  - Realistic think times between page transitions (e.g., 3–8 seconds for catalog browsing, 15–30 seconds for filling out order forms)
  - Pacing strategies that match production traffic patterns
  - Appropriate ramp-up curves based on expected user behavior

- **Script completeness review** — Ask DANA to review your JMeter test plan XML against the full list of application endpoints to identify:
  - Endpoints not covered by any sampler
  - Missing assertions or inadequate validation
  - Parameterization gaps (hardcoded values that should be dynamic)
  - Correlation candidates (dynamic values not being extracted)

**Suggested talking points:**

> "DANA can act as a second pair of eyes on your test scripts. Upload the JMX file and ask: 'Are there any JPetStore endpoints missing from this test plan?' — DANA will cross-reference against the application's endpoint inventory."

> "For think times, instead of guessing, give DANA a set of access logs and ask for the median inter-request delay per user session. You'll get data-driven pacing values."

---

## 5. Phase 3: Test Execution & Monitoring

### 5.1 What the Presenter Does

1. **Launch JPetStore** using Docker:
   ```bash
   docker-compose up -d
   ```
   Or via Maven with an embedded server profile:
   ```bash
   ./mvnw cargo:run -P tomcat90
   ```

2. **Execute JMeter tests** in non-GUI mode for accurate performance measurement:
   ```bash
   jmeter -n -t jpetstore-test-plan.jmx -l results.jtl -e -o report/
   ```

3. **Monitor during execution:**
   - JMeter real-time summary in the console (throughput, avg response time, error %)
   - Application server logs for errors and warnings
   - System resource utilization (CPU, memory, network)

4. **Collect result artifacts:**
   - JTL/CSV result files with per-request metrics
   - JMeter HTML report (generated with `-e -o`)
   - Server-side logs

### 5.2 DANA Integration Point: Real-Time Analysis

**Where DANA adds value:**

- **Live execution monitoring** — While the test runs, feed intermediate result data to DANA for rolling analysis:
  - Detect sudden latency spikes as they happen (e.g., "Response times for `/orders/newOrder` jumped from 200ms to 1,500ms at the 10-minute mark")
  - Track error rate trends in real time (e.g., "HTTP 500 errors started appearing for `/cart/addItemToCart` after 50 concurrent users")
  - Identify throughput plateaus that signal resource bottlenecks

- **Anomaly detection** — DANA can flag:
  - Requests whose response time exceeds 2x the running average
  - Error rate changes that cross warning thresholds (e.g., > 1%, > 5%)
  - Sudden drops in throughput that may indicate thread pool exhaustion or database connection limits

- **Application-database correlation** — DANA can cross-reference:
  - Slow HTTP responses with the underlying MyBatis SQL queries being executed
  - Error patterns with specific item IDs or user accounts in the test data
  - Throughput degradation with inventory update contention (the `updateInventoryQuantity` operation in `OrderService`)

**Suggested talking points:**

> "Traditional monitoring means watching a JMeter console scroll by and hoping you spot problems. DANA can watch for you and proactively surface anomalies — 'Heads up: error rates on the checkout endpoint just crossed 2%.'"

> "DANA can correlate what's happening at the HTTP layer with what's happening at the database layer, helping you identify root causes during the run rather than after."

---

## 6. Phase 4: Test Run Analysis (PRIMARY DANA Showcase)

> **This is the strongest integration point for DANA. Allocate the most demo time here.**

### 6.1 What the Presenter Does (Traditional Approach)

1. Open JMeter's HTML report and review aggregate tables
2. Manually examine response time distributions per endpoint
3. Export data to Excel or a BI tool for custom analysis
4. Spend significant time building pivot tables, charts, and cross-tabulations
5. Compare results against SLA thresholds manually

### 6.2 DANA Integration Point: Deep Performance Analysis

**DANA transforms this phase from hours of manual work into an interactive, conversational analysis session.**

#### 6.2.1 Statistical Analysis

Feed DANA the JTL/CSV result file and ask for:

- **Percentile distributions** — p50, p90, p95, p99 response times per endpoint
- **Standard deviation** — identify endpoints with inconsistent performance
- **Throughput trends over time** — requests/second plotted across the test duration
- **Error classification** — group errors by HTTP status code, endpoint, and time window
- **Apdex scores** — application performance index based on configurable thresholds

Example prompt:
> "Analyze this JTL file. Give me a table of p50, p90, p95, and p99 response times for each endpoint, sorted by p99 descending."

#### 6.2.2 Bottleneck Identification

DANA can correlate multiple data dimensions to pinpoint bottlenecks:

- **Slow transactions by endpoint** — "Which endpoint had the highest p99 latency?"
- **Time-based patterns** — "Did response times degrade as the test progressed? At what concurrency level?"
- **Data-dependent patterns** — "Were errors concentrated on specific item IDs or user accounts?"
- **Throughput vs. response time** — "At what throughput level did response times start to degrade?"

#### 6.2.3 Multi-Run Comparison

When you have results from multiple test runs (e.g., before and after a code change, or at different load levels):

- **Regression detection** — "Compare Run A (baseline) with Run B (after optimization). Which endpoints improved? Which regressed?"
- **Scalability analysis** — "Show me how response times changed from 10 users to 50 users to 100 users"
- **Statistical significance** — "Is the 15% improvement in checkout response time statistically significant?"

#### 6.2.4 Root Cause Analysis

DANA can perform deeper investigation:

- **Error pattern analysis** — "What percentage of errors occurred in the last 20% of the test? Is this indicative of a resource leak?"
- **Correlation with request parameters** — "Do orders with more than 3 line items take significantly longer?"
- **Temporal clustering** — "Are slow responses clustered at regular intervals, suggesting garbage collection pauses?"

#### 6.2.5 Visualization Generation

DANA can produce charts and graphs for:

- **Response time trends** — line charts showing p50/p90/p99 over the test duration
- **Throughput graphs** — requests/second over time with error overlays
- **Error distribution** — pie or bar charts grouped by status code and endpoint
- **Comparison heatmaps** — endpoint-vs-metric grids highlighting regressions in red and improvements in green
- **Percentile histograms** — distribution curves for response times per endpoint

#### 6.2.6 Natural Language Q&A

The most powerful DANA capability — ask questions in plain English:

| Example Question | What DANA Returns |
|---|---|
| "Which endpoint had the highest p99 latency?" | Ranked table with endpoint names and p99 values |
| "Did error rates increase over time?" | Time-series analysis with trend line and inflection points |
| "How does this run compare to the baseline?" | Side-by-side comparison table with delta percentages |
| "What was the peak throughput and when did it occur?" | Timestamp, value, and context around peak |
| "Show me the slowest 1% of requests — what do they have in common?" | Grouped analysis by endpoint, time, and parameters |
| "Is there a correlation between response time and time of day in the test?" | Scatter plot with correlation coefficient |

**Suggested talking points:**

> "What used to take an experienced performance engineer 2–4 hours in Excel now takes minutes with DANA. And the analysis is more thorough because DANA examines every data point, not just the aggregates."

> "The key differentiator is natural language interaction. You don't need to know R, Python, or pivot table formulas. Just ask your question."

> "DANA doesn't just give you numbers — it gives you context and recommendations. It identifies the 'so what' behind the data."

---

## 7. Phase 5: Performance Test Summary Report (STRONG DANA Showcase)

### 7.1 What the Presenter Does (Traditional Approach)

1. Open a report template (Word/Confluence)
2. Manually copy metrics from JMeter reports
3. Build tables and charts in Excel, paste into the document
4. Write narrative sections explaining the results
5. Formulate recommendations
6. Review, format, and distribute — often taking half a day or more

### 7.2 DANA Integration Point: Automated Report Generation

**DANA can generate a complete, publication-ready performance test summary report from raw JTL data.**

#### 7.2.1 Executive Summary

DANA automatically generates:

- One-paragraph test overview (application, environment, test dates, load profile)
- **Pass/fail evaluation** against defined SLA criteria (e.g., "p95 < 2s for all endpoints", "Error rate < 1%")
- Traffic light summary: which endpoints passed, which are at risk, which failed
- Key headline metrics (peak throughput, overall error rate, worst-performing endpoint)

#### 7.2.2 Detailed Statistics Tables

Formatted, presentation-ready tables including:

| Endpoint | Samples | Avg (ms) | p50 (ms) | p90 (ms) | p95 (ms) | p99 (ms) | Min (ms) | Max (ms) | Std Dev | Error % | Throughput (req/s) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `/catalog` | 15,230 | 45 | 38 | 82 | 105 | 210 | 12 | 1,240 | 34.2 | 0.02% | 84.6 |
| `/orders/newOrder` | 2,105 | 320 | 285 | 520 | 680 | 1,100 | 95 | 3,200 | 185.7 | 0.95% | 11.7 |
| *... (all endpoints)* | | | | | | | | | | | |

#### 7.2.3 Trend Analysis

When historical data is available, DANA generates:

- Run-over-run comparison tables showing improvement or regression percentages
- Trend charts tracking key metrics across the last N test runs
- Velocity indicators (are things getting better or worse over time?)

#### 7.2.4 Risk Highlights and Recommendations

DANA identifies and documents:

- Endpoints approaching SLA thresholds (yellow zone)
- Endpoints exceeding SLA thresholds (red zone)
- Capacity projections (e.g., "At current degradation rates, `/orders/newOrder` will breach the 2s SLA at approximately 120 concurrent users")
- Actionable recommendations (e.g., "Investigate database query performance for the order insertion path", "Consider connection pool tuning for the HSQLDB DataSource")

#### 7.2.5 Multi-Format Output

DANA can produce the report in:

- **Markdown** — for version control, wikis, and developer-friendly documentation
- **HTML** — styled and ready for browser viewing or email distribution
- **PDF-ready** — structured content suitable for PDF conversion tools

#### 7.2.6 Audience-Tailored Reporting

DANA can adjust the report's language and focus:

- **Technical team version** — includes query-level details, thread dump analysis references, configuration recommendations, and raw data appendices
- **Management version** — focuses on SLA compliance, risk assessment, business impact statements, and go/no-go recommendations with visual dashboards

**Suggested talking points:**

> "Report writing is the least favorite task of every performance tester. DANA eliminates the drudgery while producing more consistent, more thorough reports than manual efforts."

> "The ability to generate both a technical deep-dive for the engineering team and an executive summary for management — from the same data, in one session — is a game changer for stakeholder communication."

---

## 8. Summary Table

| Workshop Phase | Traditional Approach | DANA Value-Add |
|---|---|---|
| **Phase 1: Test Data Generation** | Manually write SQL scripts or use custom generators; trial-and-error to get realistic distributions | Analyzes schema and existing data; generates production-like datasets respecting all constraints; recommends volumes and distributions based on e-commerce patterns |
| **Phase 2: JMeter Script Creation** | Manually identify endpoints and build scripts; guess at think times; peer review for coverage gaps | Analyzes logs/HAR files to identify critical journeys; recommends data-driven think times; reviews scripts for endpoint coverage and parameterization completeness |
| **Phase 3: Test Execution & Monitoring** | Watch JMeter console output; react to obvious errors; limited real-time analysis | Monitors for anomalies in real time; flags latency spikes and error rate changes as they occur; correlates HTTP behavior with database query patterns |
| **Phase 4: Test Run Analysis** | Export to Excel; build pivot tables and charts manually; time-consuming and error-prone | Ingests JTL files directly; provides instant statistical analysis (percentiles, trends, comparisons); answers natural language questions; generates visualizations on demand |
| **Phase 5: Summary Report** | Copy-paste metrics into templates; manually build tables and charts; hours of formatting | Automatically generates complete, formatted reports with executive summaries, detailed statistics, trend analysis, and actionable recommendations in multiple output formats |

---

## 9. Key Talking Points for the Presenter

### Speed and Efficiency

- DANA reduces test data generation from hours to minutes by understanding schema relationships and generating constraint-aware SQL automatically.
- Post-test analysis that traditionally takes 2–4 hours in Excel can be completed in a conversational 15–20 minute DANA session.
- Report generation drops from half a day of manual work to minutes of automated production.

### Natural Language Interaction

- Performance testers can ask questions about their data in plain English — no need to learn R, Python, pandas, or complex Excel formulas.
- The conversational interface lowers the barrier to deep analysis: junior testers can perform senior-level investigations.
- Questions can be iterative — "Now show me just the checkout endpoint" or "Exclude the warm-up period and recompute."

### Handling Large Datasets

- DANA processes millions of rows of JTL data without performance degradation.
- It can cross-reference multiple result files, test runs, and data sources simultaneously.
- No more sampling or "the file was too big for Excel" limitations.

### Consistency and Repeatability

- Every DANA-generated report follows the same structure and level of rigor, eliminating the variance between different analysts.
- Analysis prompts can be saved and reused across test cycles for consistent benchmarking.
- SLA evaluation criteria are applied uniformly every time.

### Democratization of Data Analysis

- Performance testers who are not data scientists can perform sophisticated statistical analysis.
- DANA makes advanced techniques (correlation analysis, anomaly detection, significance testing) accessible through conversation.
- Teams spend less time on mechanics and more time on interpretation and decision-making.

### Integration with Existing Workflows

- DANA works with standard JMeter output formats (JTL, CSV) — no special tooling or export steps required.
- Reports can be generated in markdown for Git-based workflows, HTML for web portals, or formatted for PDF distribution.
- DANA complements (not replaces) existing monitoring and APM tools by adding an analytical layer on top.

---

## 10. Suggested Demo Script

> **Preparation:** Before the live demo, ensure you have a completed JMeter test run with a JTL result file (at least 10,000 samples recommended for meaningful analysis). Optionally prepare a second JTL file from a different run for comparison.

### Step 1: Set the Stage (2 minutes)

**Presenter says:**
> "We've just completed a 15-minute JMeter load test against JPetStore with 50 concurrent users. We have a raw JTL file with approximately 50,000 request records. Let's see how DANA can help us make sense of this data."

### Step 2: Initial Analysis (3 minutes)

**Upload the JTL file to DANA and prompt:**

> "I have a JMeter JTL result file from a performance test of JPetStore-6, an e-commerce application. Please analyze this file and give me:
> 1. Total number of requests and the test duration
> 2. Overall error rate
> 3. A summary table with p50, p90, p95, and p99 response times for each endpoint
> 4. The top 3 slowest endpoints by p99"

**What to highlight for the audience:**
- Speed of analysis (seconds, not hours)
- Structured, formatted output
- Automatic endpoint grouping

### Step 3: Deep Dive — Bottleneck Investigation (3 minutes)

**Prompt DANA:**

> "The `/orders/newOrder` endpoint has the highest p99. Can you:
> 1. Show me how its response time trended over the duration of the test
> 2. Tell me if response times got worse as concurrency increased
> 3. Check if the errors on this endpoint are clustered at any particular point in time"

**What to highlight for the audience:**
- Conversational follow-up (context is maintained)
- Time-series analysis from raw data
- Correlation between load and degradation

### Step 4: Comparison Analysis (3 minutes)

**Upload a second JTL file and prompt:**

> "Here is a second result file from a baseline run with 25 users. Compare the two runs and tell me:
> 1. Which endpoints showed the biggest regression when we doubled the load?
> 2. Did error rates increase proportionally or disproportionately?
> 3. Is the application scaling linearly?"

**What to highlight for the audience:**
- Multi-file analysis
- Regression detection
- Scalability assessment

### Step 5: Visualization (2 minutes)

**Prompt DANA:**

> "Generate a response time trend chart showing p50 and p99 for the top 5 endpoints over the test duration. Also create a throughput-vs-error-rate chart."

**What to highlight for the audience:**
- Instant chart generation
- Publication-ready visualizations
- No need for external charting tools

### Step 6: Automated Report Generation (3 minutes)

**Prompt DANA:**

> "Generate a complete performance test summary report for the 50-user run. Include:
> - Executive summary with pass/fail against these SLAs: p95 < 2 seconds, error rate < 1%
> - Detailed per-endpoint statistics table
> - Comparison with the 25-user baseline run
> - Risk highlights and recommendations
> - Format as markdown"

**What to highlight for the audience:**
- Complete report in one prompt
- SLA evaluation built in
- Professional formatting and structure
- Actionable recommendations generated automatically

### Step 7: Audience-Tailored Version (2 minutes)

**Prompt DANA:**

> "Now regenerate just the executive summary section, but tailor it for a non-technical management audience. Focus on business risk, SLA compliance, and go/no-go recommendation. Keep it under 200 words."

**What to highlight for the audience:**
- Same data, different audience
- DANA adjusts language, detail level, and focus
- Saves the team from writing two separate reports

### Step 8: Open Q&A with DANA (2 minutes)

**Invite the audience to suggest questions to ask DANA. Example prompts:**

- "What was the busiest 1-minute interval during the test?"
- "If I need to guarantee p99 < 500ms for the catalog endpoint, what's the maximum concurrent user count we can support?"
- "Which user accounts in the test data generated the most errors?"
- "Summarize the test results in 3 bullet points for a Slack message to the dev team."

**What to highlight for the audience:**
- Flexibility — any question is fair game
- DANA handles both technical depth and casual summaries
- The interactive nature makes it a powerful tool for test review meetings

---

## Appendix: Quick Reference

### JPetStore-6 Startup Commands

```bash
# Build the application
./mvnw clean package -DskipTests

# Run with Docker
docker-compose up -d

# Run with embedded Tomcat via Maven
./mvnw cargo:run -P tomcat90

# Application URL
http://localhost:8080
```

### JMeter CLI Quick Reference

```bash
# Run test in non-GUI mode
jmeter -n -t test-plan.jmx -l results.jtl

# Run test with HTML report generation
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/

# Generate report from existing JTL
jmeter -g results.jtl -o report/
```

### Default Test Credentials

| Username | Password |
|---|---|
| `j2ee` | `j2ee` |
| `ACID` | `ACID` |
| `user1` – `user5` | `pass` |

### Database Tables and Relationships

```
supplier ──────────────────────────────┐
                                       ▼
category ──► product ──► item ◄── inventory
                          │
                          ▼
signon ──► account ──► profile    bannerdata
               │
               ▼
         sequence ──► orders ──► orderstatus
                         │
                         ▼
                      lineitem
```
