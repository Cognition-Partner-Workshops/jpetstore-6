# JPetStore JMeter Performance Test Plan

## Overview

This JMeter test plan (`JPetStore_TestPlan.jmx`) implements an end-to-end user journey for the JPetStore e-commerce application. It covers the complete purchase flow from landing to logout, with proper correlation, parameterization, assertions, think times, and error handling.

## User Journey (per Virtual User)

```
1. Launch Home Page          → GET  /actions/Catalog.action
2. Navigate to Sign In       → GET  /actions/Account.action?signonForm=
3. Login (POST credentials)  → POST /actions/Account.action
4. For EACH category (FISH, DOGS, REPTILES, CATS, BIRDS):
   a. View Category          → GET  /actions/Catalog.action?viewCategory=&categoryId={cat}
   b. View First Product     → GET  /actions/Catalog.action?viewProduct=&productId={pid}
   c. Add First Item to Cart → GET  /actions/Cart.action?addItemToCart=&workingItemId={iid}
   d. Proceed to Checkout    → GET  /actions/Order.action?newOrderForm=
   e. Submit Order (Continue)→ POST /actions/Order.action (newOrder=Continue)
   f. Confirm Order          → GET  /actions/Order.action?newOrder=&confirmed=true
5. Logout                    → GET  /actions/Account.action?signoff=
```

## Test Plan Structure

### Global Configuration

| Element | Purpose |
|---------|---------|
| **UDV - Global Configuration** | Parameterized host, port, credentials, think time bounds |
| **HTTP Request Defaults** | Centralizes protocol/host/port; sets connect (10s) and response (30s) timeouts |
| **HTTP Cookie Manager** | Manages JSESSIONID; clears cookies each iteration for clean sessions |
| **HTTP Header Manager** | Sends realistic browser headers (Accept, Accept-Language, Accept-Encoding) |

### Thread Group Settings

| Parameter | Default | Notes |
|-----------|---------|-------|
| Threads (VUs) | 5 | Adjust for desired concurrency |
| Ramp-Up | 10s | Staggers VU starts |
| Loop Count | 1 | Each VU runs the full journey once |
| On Error | Start Next Loop | Skips to next iteration on failure |

### Correlation (Dynamic Values)

The Stripes framework uses hidden `_sourcePage` and `__fp` tokens in every form. These must be extracted from the GET response and submitted with the POST:

| Token | Extracted From | Used In |
|-------|---------------|---------|
| `login_sourcePage` / `login_fp` | Sign In page (Step 2) | Login POST (Step 3) |
| `order_sourcePage` / `order_fp` | Checkout form (Step 4d) | Order Continue POST (Step 4e) |

> **Note:** The Confirm Order step (4f) uses a GET link (`stripes:link`), not a form POST, so no `_sourcePage`/`__fp` tokens are needed.

### Data Extraction

| Variable | Regex/CSS | Source | Purpose |
|----------|-----------|--------|---------|
| `categoryId_*` | `viewCategory=&categoryId=([A-Z]+)` | Home page after login | All available categories |
| `productId` | `viewProduct=&productId=([^"]+)` | Category page | First product in category |
| `itemId` | `addItemToCart=&workingItemId=([^"]+)` | Product page | First item's Add to Cart ID |
| `creditCard`, `expiryDate`, `billFirstName`, etc. | Various field extractors | Checkout form | Pre-populated billing fields |
| `orderNumber` | `Order #(\d+)` | Confirmation page | For logging/validation |

### Assertions

Every transaction includes response assertions:

| Step | Assertion | Validates |
|------|-----------|-----------|
| Home Page | Contains "JPetStore Demo" | Page loaded |
| Home Page | Contains "Sign In" | Not already logged in |
| Sign In | Contains "Please enter your username and password" | Login form displayed |
| Login | Contains "Sign Out" | Login succeeded |
| Category | Contains "Product ID" | Category table loaded |
| Product | Contains "Item ID" | Product items table loaded |
| Add to Cart | Contains "Shopping Cart" | Cart page displayed |
| Add to Cart | Contains "Proceed to Checkout" | Cart is not empty |
| Checkout | Contains "Payment Details" | Checkout form loaded |
| Continue | Contains "Confirm" | Confirmation page shown |
| Confirm | Contains "Thank you" | Order placed successfully |
| Logout | Contains "Sign In" | Logged out successfully |

### Think Times

Uniform Random Timers are placed after each transaction to simulate realistic user behavior:
- **Minimum**: 1000ms (configurable via `THINK_TIME_MIN`)
- **Maximum**: 3000ms (configurable via `THINK_TIME_MAX`)

### Groovy Scripts (JSR223)

1. **Initialize Variables** - Sets up per-thread tracking; uses `SampleResult.setIgnore()` to exclude from metrics
2. **Select Random Category** - Deduplicates extracted categories, stores unique list for ForEach iteration, picks random category
3. **Log Order Completion** - Logs order number, category, and item ID after each successful order
4. **Log Journey Summary** - Logs completion statistics at end of journey

## How to Run

### GUI Mode (Development/Debugging)

```bash
jmeter -t jmeter/JPetStore_TestPlan.jmx
```

### CLI Mode (Load Testing)

```bash
jmeter -n -t jmeter/JPetStore_TestPlan.jmx \
  -JHOST=52.146.7.63 \
  -JPORT=8080 \
  -l results.jtl \
  -e -o report/
```

### Override Parameters

| Property | Default | CLI Override |
|----------|---------|-------------|
| Host | 52.146.7.63 | `-JHOST=your-host` |
| Port | 8080 | `-JPORT=your-port` |
| Username | j2ee | `-JUSERNAME=your-user` |
| Password | j2ee | `-JPASSWORD=your-pass` |
| Think Time Min | 1000 | `-JTHINK_TIME_MIN=500` |
| Think Time Max | 3000 | `-JTHINK_TIME_MAX=5000` |
| Pacing | 5000 | `-JPACING_MS=10000` |
| Results file | jmeter-results.csv | `-JresultsFile=/path/to/file.csv` |

### Scaling Up

To increase load, modify Thread Group settings in the JMX or via CLI:
```bash
jmeter -n -t jmeter/JPetStore_TestPlan.jmx \
  -Jthreads=50 -Jrampup=60 -Jduration=300
```
> Note: Thread count, ramp-up, and duration must be manually updated in the JMX file or use JMeter properties with `__P()` functions.

## Best Practices

1. **Disable View Results Tree** in load tests (high memory usage)
2. **Run in CLI mode** for actual performance tests
3. **Monitor server resources** alongside JMeter metrics
4. **Start small** (5 VUs) and gradually increase to find breaking points
5. **Review assertion failures** before scaling - they indicate script issues, not performance issues

## File Structure

```
jmeter/
├── JPetStore_TestPlan.jmx    # Main test plan
└── README.md                 # This file
```
