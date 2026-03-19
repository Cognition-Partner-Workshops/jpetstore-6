# JPetStore 6 - JMeter Performance Test Analysis Report

**Test Date:** March 17, 2026  
**Report Generated:** March 19, 2026  
**Application Under Test:** JPetStore 6 (http://52.146.7.63:8080)  
**Test Tool:** Apache JMeter  

---

## 1. Executive Summary

A JMeter load test was executed against the JPetStore 6 e-commerce application on March 17, 2026, simulating **5 concurrent virtual users** performing end-to-end shopping journeys over approximately **4.9 minutes (295.7 seconds)**.

### Key Highlights

| Metric | Value |
|--------|-------|
| **Total Requests** | 1,111 |
| **Pass Rate** | **100%** (1,111/1,111) |
| **Fail Rate** | **0.00%** (0/1,111) |
| **Overall Avg Response Time** | 70.1 ms |
| **95th Percentile Response Time** | 149.0 ms |
| **Overall Throughput** | 3.76 req/s |
| **Total Data Transferred** | 3.92 MB received / 383.5 KB sent |

**Overall Assessment: PASS** - The application handled the 5-user concurrent load with zero errors, consistent response times, and no performance degradation over the test duration. All endpoints responded within acceptable thresholds.

---

## 2. Test Scope & Configuration

### Test Parameters

| Parameter | Value |
|-----------|-------|
| **Thread Group** | JPetStore User Journey 1 |
| **Number of Threads (Virtual Users)** | 5 |
| **Test Duration** | ~4.9 minutes (295.7 seconds) |
| **Total Samples** | 1,111 |
| **Test Start Time** | 2026-03-17 09:34:55 UTC |
| **Test End Time** | 2026-03-17 09:39:50 UTC |
| **Target Server** | 52.146.7.63:8080 |
| **Loop Count** | ~7-8 iterations per thread |

### User Journey (Transaction Flow)

Each virtual user executed the following end-to-end shopping workflow:

| Step | Action | Endpoint |
|------|--------|----------|
| **Step 1** | Launch Home Page | `GET /actions/Catalog.action` |
| **Step 2** | Navigate to Sign In Page | `GET /actions/Account.action?signonForm=` |
| **Step 3** | Login (POST + Redirect) | `POST /actions/Account.action` -> `GET /actions/Catalog.action` |
| **Step 4** | Init Category Iterator | Internal logic (JSR223/BeanShell) |
| **Step 5a** | View Category (FISH, DOGS, REPTILES, CATS, BIRDS) | `GET /actions/Catalog.action?viewCategory=&categoryId={ID}` |
| **Step 5b** | View Product Detail | `GET /actions/Catalog.action?viewProduct=&productId={ID}` |
| **Step 5c** | Add Item to Cart | `GET /actions/Cart.action?addItemToCart=&workingItemId={ID}` |
| **Step 6** | View Cart | `GET /actions/Cart.action?viewCart=` |
| **Step 7** | Logout (POST + Redirect) | `POST /actions/Account.action?signoff=` -> `GET /actions/Catalog.action` |

**Products Tested:** FI-SW-01 (Angelfish), K9-BD-01 (Bulldog), RP-SN-01 (Rattlesnake), FL-DSH-01 (Manx Cat), AV-CB-01 (Amazon Parrot)  
**Cart Items Tested:** EST-1, EST-6, EST-11, EST-14, EST-18

---

## 3. Performance Metrics Summary

### 3.1 Per-Endpoint Response Time Breakdown

| Label | Total Requests | Pass | Fail | Error % | Avg (ms) | Min (ms) | Max (ms) | Median (ms) | P90 (ms) | P95 (ms) | P99 (ms) |
|-------|---------------|------|------|---------|----------|----------|----------|-------------|----------|----------|----------|
| Step 1 - Launch Home Page | 39 | 39 | 0 | 0.00% | 152.0 | 146 | 179 | 149.0 | 160.8 | 166.6 | 176.3 |
| Step 2 - Navigate to Sign In | 39 | 39 | 0 | 0.00% | 76.7 | 74 | 91 | 76.0 | 78.4 | 80.5 | 88.7 |
| Step 3 - Login (Full) | 39 | 39 | 0 | 0.00% | 155.5 | 147 | 188 | 151.0 | 171.6 | 179.7 | 187.2 |
| Step 3 - Login-0 (POST) | 39 | 39 | 0 | 0.00% | 77.9 | 73 | 100 | 75.0 | 87.0 | 87.6 | 97.3 |
| Step 3 - Login-1 (Redirect) | 39 | 39 | 0 | 0.00% | 77.3 | 73 | 93 | 75.0 | 84.4 | 87.0 | 90.7 |
| Step 4 - Init Category Iterator | 39 | 39 | 0 | 0.00% | 3.2 | 0 | 95 | 0.0 | 1.0 | 4.6 | 62.7 |
| Step 5a - View Category: BIRDS | 35 | 35 | 0 | 0.00% | 76.4 | 74 | 90 | 76.0 | 79.0 | 81.3 | 87.3 |
| Step 5a - View Category: CATS | 37 | 37 | 0 | 0.00% | 75.9 | 74 | 84 | 76.0 | 77.4 | 78.0 | 81.8 |
| Step 5a - View Category: DOGS | 37 | 37 | 0 | 0.00% | 78.1 | 74 | 89 | 76.0 | 85.4 | 86.2 | 88.3 |
| Step 5a - View Category: FISH | 39 | 39 | 0 | 0.00% | 78.0 | 74 | 90 | 76.0 | 88.0 | 89.0 | 89.6 |
| Step 5a - View Category: REPTILES | 37 | 37 | 0 | 0.00% | 77.4 | 74 | 94 | 76.0 | 83.0 | 89.0 | 92.2 |
| Step 5b - View Product: AV-CB-01 | 35 | 35 | 0 | 0.00% | 76.8 | 74 | 89 | 76.0 | 78.6 | 80.6 | 86.6 |
| Step 5b - View Product: FI-SW-01 | 38 | 38 | 0 | 0.00% | 79.1 | 75 | 94 | 77.0 | 86.6 | 91.0 | 92.9 |
| Step 5b - View Product: FL-DSH-01 | 37 | 37 | 0 | 0.00% | 76.8 | 74 | 86 | 76.0 | 79.2 | 82.2 | 84.9 |
| Step 5b - View Product: K9-BD-01 | 37 | 37 | 0 | 0.00% | 77.9 | 74 | 94 | 76.0 | 86.2 | 89.4 | 92.9 |
| Step 5b - View Product: RP-SN-01 | 37 | 37 | 0 | 0.00% | 77.3 | 75 | 101 | 76.0 | 79.0 | 79.6 | 94.2 |
| Step 5c - Add to Cart: EST-1 | 38 | 38 | 0 | 0.00% | 79.4 | 75 | 93 | 77.0 | 87.3 | 89.4 | 92.6 |
| Step 5c - Add to Cart: EST-6 | 37 | 37 | 0 | 0.00% | 79.5 | 76 | 91 | 78.0 | 85.8 | 87.6 | 90.6 |
| Step 5c - Add to Cart: EST-11 | 37 | 37 | 0 | 0.00% | 79.5 | 77 | 86 | 79.0 | 82.4 | 86.0 | 86.0 |
| Step 5c - Add to Cart: EST-14 | 35 | 35 | 0 | 0.00% | 80.7 | 78 | 98 | 80.0 | 84.2 | 86.0 | 93.9 |
| Step 5c - Add to Cart: EST-18 | 35 | 35 | 0 | 0.00% | 81.9 | 78 | 100 | 80.0 | 88.2 | 92.9 | 98.3 |
| Step 6 - View Cart | 35 | 35 | 0 | 0.00% | 80.7 | 77 | 101 | 79.0 | 82.6 | 87.6 | 96.9 |
| Step 7 - Logout (Full) | 35 | 35 | 0 | 0.00% | 152.2 | 146 | 193 | 149.0 | 164.2 | 166.2 | 184.8 |
| Step 7 - Logout-0 (POST) | 35 | 35 | 0 | 0.00% | 75.3 | 72 | 99 | 74.0 | 80.0 | 82.9 | 94.2 |
| Step 7 - Logout-1 (Redirect) | 35 | 35 | 0 | 0.00% | 76.7 | 73 | 94 | 75.0 | 82.2 | 84.0 | 90.6 |
| Set Current Category | 186 | 186 | 0 | 0.00% | 0.4 | 0 | 17 | 0.0 | 1.0 | 1.0 | 1.2 |

### 3.2 Throughput & Bandwidth

| Label | Throughput (req/s) | Total Bytes Received | Total Bytes Sent |
|-------|-------------------|---------------------|-----------------|
| Step 1 - Launch Home Page | 0.13 | 218,985 | 11,817 |
| Step 2 - Navigate to Sign In | 0.13 | 157,790 | 14,352 |
| Step 3 - Login (Full) | 0.14 | 207,948 | 40,209 |
| Step 5a - View Categories (all) | 0.70 | 735,751 | 72,593 |
| Step 5b - View Products (all) | 0.68 | 774,432 | 71,429 |
| Step 5c - Add to Cart (all) | 0.67 | 1,023,173 | 71,252 |
| Step 6 - View Cart | 0.13 | 226,485 | 12,705 |
| Step 7 - Logout (Full) | 0.13 | 181,300 | 25,235 |
| Set Current Category | 0.64 | 4,275 | 0 |
| **TOTAL** | **3.76** | **3,920,518 (3.74 MB)** | **383,507 (374.5 KB)** |

### 3.3 Overall Response Time Distribution

| Statistic | Value |
|-----------|-------|
| Average | 70.1 ms |
| Min | 0 ms |
| Max | 193 ms |
| Median | 76.0 ms |
| Std Deviation | 41.8 ms |
| 90th Percentile | 146.0 ms |
| 95th Percentile | 149.0 ms |
| 99th Percentile | 165.0 ms |

---

## 4. Failure Analysis

### 4.1 Summary

| Metric | Value |
|--------|-------|
| **Total Failures** | **0** |
| **Error Rate** | **0.00%** |
| **Failed Endpoints** | None |

**No failures were recorded during the entire test run.** All 1,111 requests across all endpoints returned successful responses.

### 4.2 HTTP Response Code Distribution

| Response Code | Count | Percentage | Description |
|--------------|-------|------------|-------------|
| **200 (OK)** | 1,037 | 93.3% | Successful page/resource delivery |
| **302 (Found/Redirect)** | 74 | 6.7% | Expected redirects during Login and Logout flows |

All response codes are within expected ranges. The 302 redirects are part of the normal Login (POST -> redirect to catalog) and Logout (signoff -> redirect to catalog) flows.

### 4.3 Failure Patterns

No failure patterns to report. The application maintained 100% availability throughout the test duration under the tested load conditions.

---

## 5. Performance Bottlenecks

### 5.1 Slowest Endpoints (by Average Response Time)

| Rank | Endpoint | Avg (ms) | P95 (ms) | P99 (ms) | Notes |
|------|----------|----------|----------|----------|-------|
| 1 | **Step 3 - Login** | 155.5 | 179.7 | 187.2 | Composite: POST + redirect (two HTTP calls) |
| 2 | **Step 7 - Logout** | 152.2 | 166.2 | 184.8 | Composite: POST + redirect (two HTTP calls) |
| 3 | **Step 1 - Launch Home Page** | 152.0 | 166.6 | 176.3 | Initial connection overhead (avg 75ms connect time) |
| 4 | **Step 5c - Add to Cart: EST-18** | 81.9 | 92.9 | 98.3 | Highest single-request response time among cart operations |
| 5 | **Step 5c - Add to Cart: EST-14** | 80.7 | 86.0 | 93.9 | Slightly elevated compared to other add-to-cart operations |

> **Note:** The Login and Logout steps show ~150ms because they are composite transactions involving two sequential HTTP requests (POST + redirect GET). When broken down individually, each sub-request is ~75-78ms, which is consistent with other single-request endpoints.

### 5.2 Connection Time Analysis

| Endpoint | Avg Connect Time (ms) |
|----------|----------------------|
| Step 1 - Launch Home Page | **75.0** |
| All other endpoints | **0.0** |

The initial "Launch Home Page" request incurs a ~75ms connection establishment overhead (TCP handshake + TLS if applicable). All subsequent requests reuse the established connection (connection pooling), resulting in 0ms connect time. This is expected behavior for HTTP keep-alive connections.

### 5.3 Latency vs. Elapsed Time

For most single-request endpoints, latency equals elapsed time, indicating minimal server processing overhead and efficient response delivery. The composite transactions (Login, Logout) show latency of ~75-78ms per individual sub-request, confirming the server processes each request efficiently.

### 5.4 Performance Over Time (30-Second Intervals)

| Time Window | Requests | Avg Response (ms) | Max Response (ms) | Failures |
|-------------|----------|-------------------|-------------------|----------|
| 0s - 30s | 104 | 72.2 | 188 | 0 |
| 30s - 60s | 114 | 75.5 | 179 | 0 |
| 60s - 90s | 109 | 69.2 | 153 | 0 |
| 90s - 120s | 108 | 70.0 | 155 | 0 |
| 120s - 150s | 127 | 66.1 | 160 | 0 |
| 150s - 180s | 119 | 69.8 | 160 | 0 |
| 180s - 210s | 100 | 67.2 | 151 | 0 |
| 210s - 240s | 115 | 68.8 | 155 | 0 |
| 240s - 270s | 105 | 67.7 | 150 | 0 |
| 270s - 300s | 110 | 74.6 | 193 | 0 |

**No performance degradation detected.** Response times remained stable throughout the test, with average response times ranging between 66.1ms and 75.5ms across all intervals. The slightly elevated initial interval (72.2ms) is attributable to connection establishment overhead for new threads.

---

## 6. Findings & Observations

### Critical

> **None** - No critical issues identified.

### High

> **None** - No high-severity issues identified.

### Medium

| # | Finding | Details |
|---|---------|---------|
| M1 | **Limited Load Level** | The test was conducted with only 5 concurrent users. This is insufficient to determine the application's breaking point, maximum capacity, or behavior under stress. Production workloads may involve significantly higher concurrency. |
| M2 | **No Order Placement Tested** | The user journey stops at "View Cart" and does not proceed to checkout/order placement. The `OrderService.insertOrder()` transactional operation (which updates inventory, inserts order records, and inserts line items) was not tested. This is a critical business flow that involves database writes and could be a bottleneck. |
| M3 | **Single User Account** | All 5 threads appear to use the same login credentials. This does not simulate realistic multi-user behavior and may mask session-related concurrency issues. |

### Low

| # | Finding | Details |
|---|---------|---------|
| L1 | **Home Page Connect Time** | Initial requests show ~75ms connection time, suggesting the server is geographically distant from the test client or there's TLS negotiation overhead. This is not a server-side issue. |
| L2 | **Cart Accumulation Pattern** | The "Add to Cart" operations for items added later in the journey (EST-14, EST-18) show slightly higher response times (~80-82ms vs ~79ms for EST-1). This is expected as the cart grows and response payloads increase. |
| L3 | **Init Category Iterator Outlier** | Step 4 shows a P99 of 62.7ms despite a median of 0ms, with one spike to 95ms. This is likely a JMeter-internal operation (JSR223/BeanShell) and not server-side, but worth monitoring. |
| L4 | **No Think Time Evident** | The test appears to execute requests back-to-back without simulating realistic user think time, which means the throughput observed (3.76 req/s) represents the upper bound for 5 users with minimal delays. |

---

## 7. Recommendations

### For Developers

| Priority | Recommendation | Rationale |
|----------|---------------|-----------|
| Medium | **Profile OrderService.insertOrder()** | Order placement involves multiple DB writes in a single transaction. Load test this path to ensure it scales under concurrent orders. |
| Low | **Review Session Management** | Validate that session-scoped beans (CartActionBean, AccountActionBean) behave correctly when the same user logs in from multiple sessions concurrently. |
| Low | **Monitor Cart Serialization** | As cart size grows, the response payload increases. Consider pagination or lazy loading if carts can become very large. |

### For Technical Architects

| Priority | Recommendation | Rationale |
|----------|---------------|-----------|
| **High** | **Conduct Higher-Load Testing** | Run tests with 50, 100, 200, and 500 concurrent users to identify the application's capacity ceiling and breaking point. Current 5-user test does not validate production readiness. |
| **High** | **Add Stress & Spike Tests** | Design tests that ramp up users rapidly (spike test) and exceed expected capacity (stress test) to understand failure modes and recovery behavior. |
| Medium | **Implement Connection Pooling Monitoring** | Monitor HSQLDB connection pool utilization under higher loads. The in-memory HSQLDB may become a bottleneck with concurrent write operations. |
| Medium | **Evaluate Horizontal Scaling** | If higher-load tests reveal bottlenecks, consider deploying behind a load balancer with multiple Tomcat instances. Test session replication/sticky sessions. |

### For QA / Performance Engineers

| Priority | Recommendation | Rationale |
|----------|---------------|-----------|
| **High** | **Expand Test Scenarios** | Add the following missing scenarios: (1) Order placement & checkout flow, (2) User registration, (3) Account editing, (4) Product search functionality. |
| **High** | **Increase Virtual User Count** | Scale from 5 to 50-500 users with proper ramp-up periods (e.g., 10 users/minute) to perform meaningful capacity testing. |
| Medium | **Add Think Time** | Introduce realistic think times (3-10 seconds between actions) to better simulate actual user behavior and get more realistic throughput numbers. |
| Medium | **Use Multiple User Accounts** | Create a CSV data set config with multiple unique user accounts to simulate realistic multi-user concurrency and avoid session conflicts. |
| Medium | **Add Response Assertions** | Beyond HTTP status code checks, add assertions to verify response content (e.g., check that cart items are correctly displayed, login redirect lands on correct page). |
| Low | **Add Custom Metrics** | Configure JMeter backend listener to send metrics to InfluxDB/Grafana for real-time monitoring during longer test runs. |
| Low | **Test With Production-Like Data** | Current HSQLDB contains minimal seed data. Test with larger data sets to identify query performance issues that may appear at scale. |

---

## 8. Appendix

### A. Raw Statistics - All Samplers

| # | Label | Count | Avg (ms) | Min (ms) | Max (ms) | Median (ms) | P90 (ms) | P95 (ms) | P99 (ms) | Error % | Throughput (req/s) | Recv KB | Sent KB |
|---|-------|-------|----------|----------|----------|-------------|----------|----------|----------|---------|--------------------|---------|---------|
| 1 | Set Current Category | 186 | 0.4 | 0 | 17 | 0.0 | 1.0 | 1.0 | 1.2 | 0.00% | 0.64 | 4.2 | 0.0 |
| 2 | Step 1 - Launch Home Page | 39 | 152.0 | 146 | 179 | 149.0 | 160.8 | 166.6 | 176.3 | 0.00% | 0.13 | 213.9 | 11.5 |
| 3 | Step 2 - Navigate to Sign In | 39 | 76.7 | 74 | 91 | 76.0 | 78.4 | 80.5 | 88.7 | 0.00% | 0.13 | 154.1 | 14.0 |
| 4 | Step 3 - Login | 39 | 155.5 | 147 | 188 | 151.0 | 171.6 | 179.7 | 187.2 | 0.00% | 0.14 | 203.1 | 39.3 |
| 5 | Step 3 - Login-0 | 39 | 77.9 | 73 | 100 | 75.0 | 87.0 | 87.6 | 97.3 | 0.00% | 0.14 | 6.9 | 25.7 |
| 6 | Step 3 - Login-1 | 39 | 77.3 | 73 | 93 | 75.0 | 84.4 | 87.0 | 90.7 | 0.00% | 0.14 | 196.2 | 13.6 |
| 7 | Step 4 - Init Category Iterator | 39 | 3.2 | 0 | 95 | 0.0 | 1.0 | 4.6 | 62.7 | 0.00% | 0.14 | 1.1 | 0.0 |
| 8 | Step 5a - View Category: BIRDS | 35 | 76.4 | 74 | 90 | 76.0 | 79.0 | 81.3 | 87.3 | 0.00% | 0.13 | 130.3 | 13.2 |
| 9 | Step 5a - View Category: CATS | 37 | 75.9 | 74 | 84 | 76.0 | 77.4 | 78.0 | 81.8 | 0.00% | 0.14 | 137.6 | 13.9 |
| 10 | Step 5a - View Category: DOGS | 37 | 78.1 | 74 | 89 | 76.0 | 85.4 | 86.2 | 88.3 | 0.00% | 0.14 | 157.5 | 13.9 |
| 11 | Step 5a - View Category: FISH | 39 | 78.0 | 74 | 90 | 76.0 | 88.0 | 89.0 | 89.6 | 0.00% | 0.14 | 155.2 | 14.7 |
| 12 | Step 5a - View Category: REPTILES | 37 | 77.4 | 74 | 94 | 76.0 | 83.0 | 89.0 | 92.2 | 0.00% | 0.14 | 137.8 | 14.1 |
| 13 | Step 5b - View Product: AV-CB-01 | 35 | 76.8 | 74 | 89 | 76.0 | 78.6 | 80.6 | 86.6 | 0.00% | 0.13 | 136.2 | 13.3 |
| 14 | Step 5b - View Product: FI-SW-01 | 38 | 79.1 | 75 | 94 | 77.0 | 86.6 | 91.0 | 92.9 | 0.00% | 0.13 | 157.8 | 14.4 |
| 15 | Step 5b - View Product: FL-DSH-01 | 37 | 76.8 | 74 | 86 | 76.0 | 79.2 | 82.2 | 84.9 | 0.00% | 0.14 | 153.7 | 14.1 |
| 16 | Step 5b - View Product: K9-BD-01 | 37 | 77.9 | 74 | 94 | 76.0 | 86.2 | 89.4 | 92.9 | 0.00% | 0.14 | 153.9 | 14.0 |
| 17 | Step 5b - View Product: RP-SN-01 | 37 | 77.3 | 75 | 101 | 76.0 | 79.0 | 79.6 | 94.2 | 0.00% | 0.14 | 154.7 | 14.0 |
| 18 | Step 5c - Add to Cart: EST-1 | 38 | 79.4 | 75 | 93 | 77.0 | 87.3 | 89.4 | 92.6 | 0.00% | 0.13 | 178.7 | 14.4 |
| 19 | Step 5c - Add to Cart: EST-6 | 37 | 79.5 | 76 | 91 | 78.0 | 85.8 | 87.6 | 90.6 | 0.00% | 0.14 | 188.8 | 14.0 |
| 20 | Step 5c - Add to Cart: EST-11 | 37 | 79.5 | 77 | 86 | 79.0 | 82.4 | 86.0 | 86.0 | 0.00% | 0.14 | 203.8 | 14.1 |
| 21 | Step 5c - Add to Cart: EST-14 | 35 | 80.7 | 78 | 98 | 80.0 | 84.2 | 86.0 | 93.9 | 0.00% | 0.13 | 206.8 | 13.3 |
| 22 | Step 5c - Add to Cart: EST-18 | 35 | 81.9 | 78 | 100 | 80.0 | 88.2 | 92.9 | 98.3 | 0.00% | 0.13 | 221.2 | 13.3 |
| 23 | Step 6 - View Cart | 35 | 80.7 | 77 | 101 | 79.0 | 82.6 | 87.6 | 96.9 | 0.00% | 0.13 | 221.2 | 12.4 |
| 24 | Step 7 - Logout | 35 | 152.2 | 146 | 193 | 149.0 | 164.2 | 166.2 | 184.8 | 0.00% | 0.13 | 177.1 | 24.6 |
| 25 | Step 7 - Logout-0 | 35 | 75.3 | 72 | 99 | 74.0 | 80.0 | 82.9 | 94.2 | 0.00% | 0.13 | 6.2 | 12.5 |
| 26 | Step 7 - Logout-1 | 35 | 76.7 | 73 | 94 | 75.0 | 82.2 | 84.0 | 90.6 | 0.00% | 0.13 | 170.9 | 12.2 |
| | **TOTAL** | **1,111** | **70.1** | **0** | **193** | **76.0** | **146.0** | **149.0** | **165.0** | **0.00%** | **3.76** | **3,828.6** | **374.5** |

### B. Response Code Summary

| Code | Meaning | Count | Percentage |
|------|---------|-------|------------|
| 200 | OK | 1,037 | 93.3% |
| 302 | Found (Redirect) | 74 | 6.7% |
| **Total** | | **1,111** | **100%** |

### C. Thread Concurrency

| Metric | Value |
|--------|-------|
| Max Group Threads | 5 |
| Max All Threads | 5 |
| Thread Group Name | JPetStore User Journey 1 |
| Threads | JPetStore User Journey 1-1 through 1-5 |

### D. Test Environment

| Component | Details |
|-----------|---------|
| Application | JPetStore 6 |
| Server Address | 52.146.7.63:8080 |
| Application Server | Apache Tomcat (inferred from URL patterns) |
| Database | HSQLDB (in-memory, embedded) |
| Framework Stack | MyBatis 3 + Spring 5 + Stripes |
| Test Tool | Apache JMeter |

---

*This report was generated by analyzing the JMeter JTL results file (`results.jtl`) containing 1,111 samples collected on March 17, 2026.*
