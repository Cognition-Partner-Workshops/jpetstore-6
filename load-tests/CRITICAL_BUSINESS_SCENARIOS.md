# JPetStore 6 - Critical Business Scenarios for Load Testing

## Table of Contents

- [1. Application Architecture Overview](#1-application-architecture-overview)
- [2. Database Schema Summary](#2-database-schema-summary)
- [3. Critical Business Scenarios](#3-critical-business-scenarios)
  - [Scenario 1: Homepage / Main Catalog View](#scenario-1-homepage--main-catalog-view)
  - [Scenario 2: Browse Category -> Product -> Item](#scenario-2-browse-category---product---item)
  - [Scenario 3: Search Products](#scenario-3-search-products)
  - [Scenario 4: User Sign-In / Authentication](#scenario-4-user-sign-in--authentication)
  - [Scenario 5: User Registration](#scenario-5-user-registration)
  - [Scenario 6: Update User Profile](#scenario-6-update-user-profile)
  - [Scenario 7: Add Item to Cart and View Cart](#scenario-7-add-item-to-cart-and-view-cart)
  - [Scenario 8: Update Cart Quantities](#scenario-8-update-cart-quantities)
  - [Scenario 9: Remove Item from Cart](#scenario-9-remove-item-from-cart)
  - [Scenario 10: Full Purchase Flow (End-to-End)](#scenario-10-full-purchase-flow-end-to-end)
  - [Scenario 11: View Order History](#scenario-11-view-order-history)
  - [Scenario 12: View Order Details](#scenario-12-view-order-details)
  - [Scenario 13: User Sign-Out](#scenario-13-user-sign-out)
- [4. Handler Method Coverage Matrix](#4-handler-method-coverage-matrix)
- [5. Complete SQL Operations Inventory](#5-complete-sql-operations-inventory)
- [6. Transactional Behavior Analysis](#6-transactional-behavior-analysis)
- [7. MyBatis Cache Configuration](#7-mybatis-cache-configuration)
- [8. Summary Table](#8-summary-table)

---

## 1. Application Architecture Overview

JPetStore 6 is a three-tier Java web application:

| Layer | Technology | Location |
|-------|-----------|----------|
| **Presentation** | Stripes Framework + JSP | `web/actions/` + `WEB-INF/jsp/` |
| **Business** | Spring 5 Services | `service/` |
| **Data** | MyBatis 3 Mappers | `mapper/` + `mapper/*.xml` |
| **Database** | HSQLDB (embedded) | `database/*.sql` |
| **Server** | Tomcat 9.0 (WAR) | `pom.xml` profiles |

### URL Routing

All requests are routed through the Stripes `DispatcherServlet` mapped to `*.action`. The `StripesFilter` scans `org.mybatis.jpetstore.web` for ActionBeans. URL patterns follow the convention:

```
/actions/{BeanName}.action?{eventName}=&{parameters}
```

### ActionBean URL Mappings

Stripes auto-generates URL bindings based on class names (no explicit `@UrlBinding` annotations are used):

| ActionBean | Auto-generated URL | Scope |
|---|---|---|
| `CatalogActionBean` | `/actions/Catalog.action` | `@SessionScope` |
| `AccountActionBean` | `/actions/Account.action` | `@SessionScope` |
| `CartActionBean` | `/actions/Cart.action` | `@SessionScope` |
| `OrderActionBean` | `/actions/Order.action` | `@SessionScope` |

> **Note:** All ActionBeans are `@SessionScope`, meaning each user's HTTP session holds a dedicated instance. This impacts memory under load and means session state accumulates across requests.

### JSP Views Inventory

| Directory | Pages | Purpose |
|---|---|---|
| `catalog/` | `Main.jsp`, `Category.jsp`, `Product.jsp`, `Item.jsp`, `SearchProducts.jsp` | Catalog browsing |
| `account/` | `SignonForm.jsp`, `NewAccountForm.jsp`, `EditAccountForm.jsp`, `IncludeAccountFields.jsp` | Account management |
| `cart/` | `Cart.jsp`, `Checkout.jsp`, `IncludeMyList.jsp` | Shopping cart |
| `order/` | `NewOrderForm.jsp`, `ShippingForm.jsp`, `ConfirmOrder.jsp`, `ViewOrder.jsp`, `ListOrders.jsp` | Order processing |
| `common/` | `IncludeTop.jsp`, `IncludeBottom.jsp`, `Error.jsp` | Shared layout |

---

## 2. Database Schema Summary

### Tables and Relationships

```
SUPPLIER (suppid PK)
    |
CATEGORY (catid PK)
    |
PRODUCT (productid PK, category FK -> CATEGORY)
    |
ITEM (itemid PK, productid FK -> PRODUCT, supplier FK -> SUPPLIER)
    |
INVENTORY (itemid PK) -- 1:1 with ITEM, stores quantity

SIGNON (username PK) -- authentication credentials
ACCOUNT (userid PK)  -- user profile data
PROFILE (userid PK)  -- user preferences (language, fav category, etc.)
BANNERDATA (favcategory PK) -- banner images per category

ORDERS (orderid PK, userid) -- order header
ORDERSTATUS (orderid + linenum composite PK) -- order status tracking
LINEITEM (orderid + linenum composite PK, itemid) -- order line items

SEQUENCE (name PK, nextid) -- ID generation for orders
```

### Seed Data Volume

| Table | Row Count | Notes |
|---|---|---|
| CATEGORY | 5 | FISH, DOGS, REPTILES, CATS, BIRDS |
| PRODUCT | 16 | Distributed across 5 categories |
| ITEM | 28 | Multiple items per product |
| INVENTORY | 28 | Each item starts with qty=10,000 |
| SUPPLIER | 2 | XYZ Pets, ABC Pets |
| SIGNON/ACCOUNT/PROFILE | 7 | Pre-loaded test users |
| BANNERDATA | 5 | One per category |
| SEQUENCE | 1 | `ordernum` starting at 1000 |

---

## 3. Critical Business Scenarios

---

### Scenario 1: Homepage / Main Catalog View

- **Criticality**: **High**
- **Category**: High Frequency User Access
- **Type**: Read-only

#### User Flow Description
1. User navigates to the application root URL (`/jpetstore/`)
2. The welcome page redirects to the main catalog view
3. The Main.jsp page renders with category sidebar links (FISH, DOGS, REPTILES, CATS, BIRDS) and a central welcome area

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Catalog.action` | `CatalogActionBean.viewMain()` (`@DefaultHandler`) |

#### Service Methods Called
- None directly (the main page is a static forward to `Main.jsp`)
- The shared `IncludeTop.jsp` header renders category sidebar links which are hardcoded HTML, not dynamic DB calls

#### Database Operations
- **None** - The main page is a simple JSP forward with no database interaction

#### Performance Risk Factors
- This is the **entry point** for all users; highest traffic endpoint
- Although no DB calls occur on this page, session creation overhead exists because `CatalogActionBean` is `@SessionScope` -- a new session-scoped bean is instantiated per new visitor
- JSP compilation on first access may cause initial latency
- The `IncludeTop.jsp` and `IncludeBottom.jsp` are included on every page, adding rendering overhead

#### Estimated User Traffic Weight
**25-30%** of total traffic (landing page, bookmarks, navigation home)

---

### Scenario 2: Browse Category -> Product -> Item

- **Criticality**: **High**
- **Category**: High Frequency User Access, Database Read Operations
- **Type**: Read-only (multi-step)

#### User Flow Description
1. User clicks a category link (e.g., "Fish") from the sidebar or main page
2. Category page displays a list of products in that category
3. User clicks a specific product (e.g., "Angelfish")
4. Product page displays a list of available items (variants) for that product
5. User clicks a specific item (e.g., "EST-1") to view item details with price and stock info

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Catalog.action?viewCategory=&categoryId=FISH` | `CatalogActionBean.viewCategory()` |
| 2 | `GET /actions/Catalog.action?viewProduct=&productId=FI-SW-01` | `CatalogActionBean.viewProduct()` |
| 3 | `GET /actions/Catalog.action?viewItem=&itemId=EST-1` | `CatalogActionBean.viewItem()` |

#### Service Methods Called
| Step | Service Method | Operation Type |
|---|---|---|
| viewCategory | `CatalogService.getProductListByCategory(categoryId)` | READ |
| viewCategory | `CatalogService.getCategory(categoryId)` | READ |
| viewProduct | `CatalogService.getItemListByProduct(productId)` | READ |
| viewProduct | `CatalogService.getProduct(productId)` | READ |
| viewItem | `CatalogService.getItem(itemId)` | READ |

#### Database Operations
| Step | SQL Operation | Table(s) | Query Details |
|---|---|---|---|
| viewCategory | `SELECT` | `CATEGORY` | Single row by CATID PK |
| viewCategory | `SELECT` | `PRODUCT` | List by CATEGORY column (indexed: `productCat`) |
| viewProduct | `SELECT` | `PRODUCT` | Single row by PRODUCTID PK |
| viewProduct | `SELECT` | `ITEM` JOIN `PRODUCT` | Join across ITEM and PRODUCT tables on PRODUCTID; returns item list with embedded product info |
| viewItem | `SELECT` | `ITEM` JOIN `INVENTORY` JOIN `PRODUCT` | **3-table join** on ITEMID and PRODUCTID; returns item with quantity and product info |

#### Performance Risk Factors
- **3-table join** on `viewItem`: `ITEM`, `INVENTORY`, `PRODUCT` joined simultaneously
- **2-table join** on `viewProduct` (getItemListByProduct): `ITEM` and `PRODUCT` implicit join (`FROM ITEM I, PRODUCT P WHERE P.PRODUCTID = I.PRODUCTID`)
- This is a sequential funnel: each step generates a new DB query, meaning a single user browsing creates 5 SELECT queries across the funnel
- `CategoryMapper` and `ProductMapper` use `<cache />` which helps with repeated lookups, but `ItemMapper` also uses `<cache />` which may serve stale inventory data
- Under high concurrency, the sequential nature of browsing amplifies DB read load proportionally

#### Estimated User Traffic Weight
**30-35%** of total traffic (core browsing experience; most users browse multiple categories/products)

---

### Scenario 3: Search Products

- **Criticality**: **Medium**
- **Category**: High Frequency User Access, Database Read Operations
- **Type**: Read-only

#### User Flow Description
1. User enters a keyword in the search box (present in the header on every page)
2. System searches for products matching the keyword
3. Search results page displays matching products

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Catalog.action?searchProducts=&keyword=fish` | `CatalogActionBean.searchProducts()` |

#### Service Methods Called
| Service Method | Operation Type |
|---|---|
| `CatalogService.searchProductList(keyword)` | READ |

#### Database Operations
| SQL Operation | Table(s) | Query Details |
|---|---|---|
| `SELECT` | `PRODUCT` | `WHERE lower(name) LIKE '%keyword%'` -- **wildcard LIKE query** |

#### Performance Risk Factors
- **LIKE query with leading wildcard** (`%keyword%`): This **cannot use the `productName` index** effectively, forcing a full table scan on the PRODUCT table
- **Multi-keyword splitting**: The service splits search input on whitespace (`keywords.split("\\s+")`) and executes a **separate SQL query for each keyword**. A search like "golden retriever" generates **2 separate LIKE queries** and merges results in Java
- **No deduplication**: The `ArrayList` merge in `searchProductList()` can return duplicate products if multiple keywords match the same product
- The `lower(name)` function call on every row prevents index usage even without the leading wildcard
- Under load, multi-word searches multiply the DB hit count linearly

#### Estimated User Traffic Weight
**5-8%** of total traffic (not all users search; many browse by category)

---

### Scenario 4: User Sign-In / Authentication

- **Criticality**: **High**
- **Category**: High Frequency User Access, Database Read Operations
- **Type**: Read-only

#### User Flow Description
1. User clicks "Sign In" link
2. Sign-on form is displayed
3. User enters username and password, submits
4. System authenticates against database
5. On success: user is redirected to main catalog with personalized content (favorite category product list)
6. On failure: error message displayed, user stays on sign-on form

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Account.action` | `AccountActionBean.signonForm()` (`@DefaultHandler`) |
| 2 | `POST /actions/Account.action?signon=` | `AccountActionBean.signon()` |
| 3 (success) | `REDIRECT /actions/Catalog.action` | `CatalogActionBean.viewMain()` |

#### Service Methods Called
| Step | Service Method | Operation Type |
|---|---|---|
| signon | `AccountService.getAccount(username, password)` | READ |
| signon (success) | `CatalogService.getProductListByCategory(favCategoryId)` | READ |

#### Database Operations
| SQL Operation | Table(s) | Query Details |
|---|---|---|
| `SELECT` | `ACCOUNT` JOIN `PROFILE` JOIN `SIGNON` JOIN `BANNERDATA` | **4-table join** to authenticate and load full user profile in one query |
| `SELECT` | `PRODUCT` | Products filtered by favourite category |

#### Performance Risk Factors
- **4-table implicit join** (`FROM ACCOUNT, PROFILE, SIGNON, BANNERDATA`): This is one of the most complex queries in the application, joining across ACCOUNT, PROFILE, SIGNON, and BANNERDATA tables with 4 WHERE conditions
- Authentication is a **gateway operation** -- every user session begins here, creating a burst of auth queries at peak times
- Successful sign-in immediately triggers a second query for the user's favorite category product list
- **Session attribute storage**: On success, the `AccountActionBean` instance is stored in the HTTP session under key `"accountBean"` (and also as `/actions/Account.action`), which increases session memory footprint
- No password hashing is evident (plaintext comparison in SQL), which is a security concern but not a performance bottleneck
- Failed login attempts still execute the full 4-table join query

#### Estimated User Traffic Weight
**8-10%** of total traffic (every authenticated session starts here)

---

### Scenario 5: User Registration

- **Criticality**: **Medium**
- **Category**: Database Write Operations
- **Type**: Write (transactional)

#### User Flow Description
1. User clicks "Register Now!" link
2. Registration form is displayed
3. User fills in account details (username, password, name, address, preferences)
4. User submits the form
5. System creates the account, profile, and sign-on records
6. User is automatically authenticated and redirected to catalog

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Account.action?newAccountForm=` | `AccountActionBean.newAccountForm()` |
| 2 | `POST /actions/Account.action?newAccount=` | `AccountActionBean.newAccount()` |
| 3 | `REDIRECT /actions/Catalog.action` | `CatalogActionBean.viewMain()` |

#### Service Methods Called
| Step | Service Method | Operation Type | Transactional? |
|---|---|---|---|
| newAccount | `AccountService.insertAccount(account)` | WRITE | **Yes (`@Transactional`)** |
| newAccount | `AccountService.getAccount(username)` | READ | No |
| newAccount | `CatalogService.getProductListByCategory(favCategoryId)` | READ | No |

#### Database Operations
| SQL Operation | Table(s) | Query Details |
|---|---|---|
| `INSERT` | `ACCOUNT` | Insert new user record (email, name, address, phone) |
| `INSERT` | `PROFILE` | Insert user preferences (language, fav category, list/banner options) |
| `INSERT` | `SIGNON` | Insert credentials (username, password) |
| `SELECT` | `ACCOUNT` JOIN `PROFILE` JOIN `SIGNON` JOIN `BANNERDATA` | 4-table join to reload full account |
| `SELECT` | `PRODUCT` | Products by favourite category |

#### Performance Risk Factors
- **`@Transactional` on `insertAccount()`**: Three sequential INSERT operations within a single transaction. If any fails, all roll back
- The 3 INSERTs must complete atomically before the transaction commits, holding DB locks on ACCOUNT, PROFILE, and SIGNON tables simultaneously
- After registration, two additional SELECT queries execute (account reload + products), adding to total response time
- Under concurrent registration load, INSERT contention on the ACCOUNT/SIGNON tables could create lock waits
- Username uniqueness is enforced by the SIGNON table's PK constraint, which means duplicate usernames will cause a transaction rollback exception

#### Estimated User Traffic Weight
**1-2%** of total traffic (registration is a one-time event per user)

---

### Scenario 6: Update User Profile

- **Criticality**: **Medium**
- **Category**: Database Read/Write Operations
- **Type**: Mixed (read + write, transactional)

#### User Flow Description
1. Authenticated user clicks "My Account" link
2. Edit account form is displayed with current account data pre-populated
3. User modifies fields and submits
4. System updates account, profile, and optionally signon records
5. User is redirected to catalog

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Account.action?editAccountForm=` | `AccountActionBean.editAccountForm()` |
| 2 | `POST /actions/Account.action?editAccount=` | `AccountActionBean.editAccount()` |
| 3 | `REDIRECT /actions/Catalog.action` | `CatalogActionBean.viewMain()` |

#### Service Methods Called
| Step | Service Method | Operation Type | Transactional? |
|---|---|---|---|
| editAccount | `AccountService.updateAccount(account)` | WRITE | **Yes (`@Transactional`)** |
| editAccount | `AccountService.getAccount(username)` | READ | No |
| editAccount | `CatalogService.getProductListByCategory(favCategoryId)` | READ | No |

#### Database Operations
| SQL Operation | Table(s) | Query Details |
|---|---|---|
| `UPDATE` | `ACCOUNT` | Update email, name, address, phone by USERID |
| `UPDATE` | `PROFILE` | Update language, fav category, list/banner options by USERID |
| `UPDATE` (conditional) | `SIGNON` | Update password only if a new password was provided (non-empty check) |
| `SELECT` | `ACCOUNT` JOIN `PROFILE` JOIN `SIGNON` JOIN `BANNERDATA` | 4-table join to reload full account |
| `SELECT` | `PRODUCT` | Products by new favourite category |

#### Performance Risk Factors
- **`@Transactional` on `updateAccount()`**: 2-3 UPDATE operations in one transaction (ACCOUNT + PROFILE, optionally SIGNON)
- The conditional password update uses `Optional.ofNullable(password).filter(p -> p.length() > 0).ifPresent(...)`, adding a third UPDATE only when password changes
- Row-level locks held on ACCOUNT and PROFILE tables during the transaction
- The subsequent 4-table join SELECT to reload the account adds latency after the writes complete
- Form pre-population relies on the session-cached `account` object (no separate DB read for the edit form)

#### Estimated User Traffic Weight
**1-2%** of total traffic (occasional profile edits)

---

### Scenario 7: Add Item to Cart and View Cart

- **Criticality**: **High**
- **Category**: High Frequency User Access, Database Read Operations
- **Type**: Read-only (cart is session-stored, not DB-persisted)

#### User Flow Description
1. User browses to an item detail page
2. User clicks "Add to Cart" button
3. System checks if item already in cart; if so, increments quantity
4. If new item: checks inventory stock status, fetches item details from DB, adds to session cart
5. Cart page is displayed

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `POST /actions/Cart.action?addItemToCart=&workingItemId=EST-1` | `CartActionBean.addItemToCart()` |

#### Service Methods Called
| Condition | Service Method | Operation Type |
|---|---|---|
| Item already in cart | None (increments in-memory quantity) | N/A |
| New item | `CatalogService.isItemInStock(itemId)` | READ |
| New item | `CatalogService.getItem(itemId)` | READ |

#### Database Operations
| Condition | SQL Operation | Table(s) | Query Details |
|---|---|---|---|
| New item | `SELECT QTY FROM INVENTORY WHERE ITEMID = ?` | `INVENTORY` | Single row lookup by PK |
| New item | `SELECT ... FROM ITEM I, INVENTORY V, PRODUCT P WHERE ...` | `ITEM` JOIN `INVENTORY` JOIN `PRODUCT` | 3-table join to get full item details |
| Existing item | None | N/A | In-memory operation only |

#### Performance Risk Factors
- For new items, **two DB queries** are executed: one for stock check and one for full item retrieval (the 3-table join)
- The stock check and item fetch could be combined into one query for efficiency, but aren't -- this is a potential optimization target
- The **Cart is stored entirely in the HTTP session** (not in the database), using a dual data structure: `HashMap<String, CartItem>` for O(1) lookup + `ArrayList<CartItem>` for ordered iteration. Under heavy add-to-cart load, session memory grows
- Adding the same item repeatedly is O(1) (HashMap lookup + quantity increment), but adding different items always triggers 2 DB queries
- The `isItemInStock` check uses the potentially-cached `ItemMapper` result, which may return stale inventory data under concurrent purchases

#### Estimated User Traffic Weight
**10-12%** of total traffic (users frequently add items while browsing)

---

### Scenario 8: Update Cart Quantities

- **Criticality**: **Low**
- **Category**: High Frequency User Access
- **Type**: Session-only (no DB operations)

#### User Flow Description
1. User views cart page
2. User modifies quantity fields for one or more items
3. User clicks "Update Cart" button
4. System parses quantity values from request parameters and updates session cart
5. Items with quantity < 1 are removed from cart

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `POST /actions/Cart.action?updateCartQuantities=` | `CartActionBean.updateCartQuantities()` |

#### Service Methods Called
- **None** - All operations are performed on the in-memory session `Cart` object

#### Database Operations
- **None** - This is a purely session-based operation

#### Performance Risk Factors
- Iterates over all cart items and parses request parameters by item ID -- potential for `NumberFormatException` (silently caught)
- No DB interaction means this is very fast, but under load the session serialization/deserialization overhead applies
- The `Iterator.remove()` pattern used for removing items with qty < 1 is safe but requires careful handling under concurrent access to the session

#### Estimated User Traffic Weight
**2-3%** of total traffic

---

### Scenario 9: Remove Item from Cart

- **Criticality**: **Low**
- **Category**: High Frequency User Access
- **Type**: Session-only (no DB operations)

#### User Flow Description
1. User views cart page
2. User clicks "Remove" link next to an item
3. System removes the item from the session cart
4. Cart page is re-displayed

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Cart.action?removeItemFromCart=&workingItemId=EST-1` | `CartActionBean.removeItemFromCart()` |

#### Service Methods Called
- **None**

#### Database Operations
- **None**

#### Performance Risk Factors
- Minimal -- O(1) HashMap removal + O(n) ArrayList removal
- Error handling for null item removal forwards to error page

#### Estimated User Traffic Weight
**1-2%** of total traffic

---

### Scenario 10: Full Purchase Flow (End-to-End)

- **Criticality**: **Critical (Highest)**
- **Category**: High Frequency User Access, Database Read/Write Operations
- **Type**: Mixed (read + heavy write, transactional)

#### User Flow Description
1. User signs in (if not already authenticated)
2. User browses catalog and adds item(s) to cart
3. User views cart and proceeds to checkout
4. Checkout page is displayed (cart summary)
5. User clicks "Proceed to Checkout" -> redirected to sign-in if not authenticated
6. New order form pre-populated with account billing/shipping info
7. User fills payment details, optionally checks "Ship to different address"
8. If different shipping address: shipping form is displayed, user fills it
9. Order confirmation page displayed with full order summary
10. User confirms order
11. System creates order: generates order ID from sequence, updates inventory for each item, inserts order, inserts order status, inserts line items
12. Cart is cleared, order confirmation displayed

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `POST /actions/Account.action?signon=` | `AccountActionBean.signon()` |
| 2 | `POST /actions/Cart.action?addItemToCart=` | `CartActionBean.addItemToCart()` |
| 3 | `GET /actions/Cart.action?viewCart=` | `CartActionBean.viewCart()` |
| 4 | `GET /actions/Cart.action?checkOut=` | `CartActionBean.checkOut()` |
| 5 | `GET /actions/Order.action?newOrderForm=` | `OrderActionBean.newOrderForm()` |
| 6 | `POST /actions/Order.action?newOrder=` (with `shippingAddressRequired=true`) | `OrderActionBean.newOrder()` -> forwards to `ShippingForm.jsp` |
| 7 | `POST /actions/Order.action?newOrder=` (shipping submitted) | `OrderActionBean.newOrder()` -> forwards to `ConfirmOrder.jsp` |
| 8 | `POST /actions/Order.action?newOrder=` (with `confirmed=true`) | `OrderActionBean.newOrder()` -> **inserts order** -> forwards to `ViewOrder.jsp` |

#### Service Methods Called
| Step | Service Method | Operation Type | Transactional? |
|---|---|---|---|
| signon | `AccountService.getAccount(user, pass)` | READ | No |
| signon | `CatalogService.getProductListByCategory(favCat)` | READ | No |
| addItemToCart | `CatalogService.isItemInStock(itemId)` | READ | No |
| addItemToCart | `CatalogService.getItem(itemId)` | READ | No |
| newOrder (confirm) | `OrderService.insertOrder(order)` | **WRITE** | **Yes (`@Transactional`)** |

#### Database Operations (Order Insertion - the critical path)
| Sequence | SQL Operation | Table(s) | Query Details |
|---|---|---|---|
| 1 | `SELECT` | `SEQUENCE` | Get current `nextId` for "ordernum" |
| 2 | `UPDATE` | `SEQUENCE` | Increment `nextId` by 1 |
| 3 | `UPDATE` (per line item) | `INVENTORY` | `SET QTY = QTY - #{increment}` for each item in the order |
| 4 | `INSERT` | `ORDERS` | Insert order header (25 columns: addresses, payment, totals) |
| 5 | `INSERT` | `ORDERSTATUS` | Insert initial order status record |
| 6 | `INSERT` (per line item) | `LINEITEM` | Insert each line item (orderId, lineNum, itemId, qty, price) |

#### Performance Risk Factors
- **This is the most critical scenario for load testing** -- it combines the most database operations into a single transactional flow
- **`@Transactional` on `insertOrder()`**: The entire order creation is atomic. This single transaction performs:
  - 1 SELECT + 1 UPDATE on SEQUENCE (ID generation with **read-modify-write pattern** -- a concurrency bottleneck)
  - N UPDATEs on INVENTORY (one per line item -- **row-level locks on inventory**)
  - 1 INSERT on ORDERS
  - 1 INSERT on ORDERSTATUS
  - N INSERTs on LINEITEM
  - Total: **2 + N + 2 + N = 4 + 2N** SQL operations per order (where N = number of items)
- **Sequence ID generation is a critical bottleneck**: `getNextId()` performs a SELECT then UPDATE on the SEQUENCE table. Under concurrent order placement, this creates a **serialization point** -- only one order can get an ID at a time, as the row-level lock on `name='ordernum'` forces sequential access
- **Inventory updates are per-item**: For an order with 5 items, 5 separate UPDATE statements execute sequentially within the transaction, each locking a row in INVENTORY
- **Transaction duration**: With N items, the transaction holds locks across SEQUENCE + INVENTORY + ORDERS + ORDERSTATUS + LINEITEM tables simultaneously. Long transactions under concurrency = deadlock risk
- **No optimistic locking**: Inventory decrements use `QTY = QTY - #{increment}` without checking if QTY would go negative. Under concurrent purchases of the same item, inventory could go negative
- **Cart clearing**: After order insertion, `cartBean.clear()` is called to reset the session cart. This is a session write operation
- The multi-step state machine in `newOrder()` (shipping -> confirmation -> insert) relies on boolean flags (`shippingAddressRequired`, `confirmed`) stored in the session-scoped bean, which means the user must follow the exact step sequence

#### Estimated User Traffic Weight
**3-5%** of total traffic (conversion rate from browsing to purchase; but this is the **most resource-intensive** scenario per-request)

---

### Scenario 11: View Order History

- **Criticality**: **Medium**
- **Category**: Database Read Operations
- **Type**: Read-only

#### User Flow Description
1. Authenticated user clicks "My Orders" link
2. System retrieves all orders for the current user
3. Order list page displays order summaries

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Order.action?listOrders=` | `OrderActionBean.listOrders()` |

#### Service Methods Called
| Service Method | Operation Type |
|---|---|
| `OrderService.getOrdersByUsername(username)` | READ |

#### Database Operations
| SQL Operation | Table(s) | Query Details |
|---|---|---|
| `SELECT` | `ORDERS` JOIN `ORDERSTATUS` | Join on ORDERID, filtered by USERID, **ordered by ORDERDATE** |

#### Performance Risk Factors
- **2-table join** with `ORDER BY ORDERDATE` -- requires sorting, which can be expensive for users with many orders
- The query fetches ALL orders for a user with no pagination -- for power users or load test scenarios that generate many orders, this could return increasingly large result sets
- The `accountBean` is retrieved from the session via `session.getAttribute("/actions/Account.action")` -- if the session is invalid or the user isn't authenticated, this will throw a NullPointerException (no null check before `getUsername()`)

#### Estimated User Traffic Weight
**2-3%** of total traffic

---

### Scenario 12: View Order Details

- **Criticality**: **Medium**
- **Category**: Database Read Operations
- **Type**: Read-only (but transactional)

#### User Flow Description
1. From order history list, user clicks on a specific order ID
2. System retrieves full order details including all line items with current inventory status
3. Order detail page displays complete order information

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Order.action?viewOrder=&orderId=1000` | `OrderActionBean.viewOrder()` |

#### Service Methods Called
| Service Method | Operation Type | Transactional? |
|---|---|---|
| `OrderService.getOrder(orderId)` | READ | **Yes (`@Transactional`)** |

#### Database Operations
| Sequence | SQL Operation | Table(s) | Query Details |
|---|---|---|---|
| 1 | `SELECT` | `ORDERS` JOIN `ORDERSTATUS` | Full order header with status, joined on ORDERID |
| 2 | `SELECT` | `LINEITEM` | All line items by ORDERID |
| 3 (per line item) | `SELECT` | `ITEM` JOIN `INVENTORY` JOIN `PRODUCT` | 3-table join to get item details |
| 4 (per line item) | `SELECT` | `INVENTORY` | Current stock quantity |

#### Performance Risk Factors
- **`@Transactional` on `getOrder()`**: Even though this is a read-only operation, it's marked transactional (ensuring consistent reads across multiple queries)
- **N+1 query problem**: For an order with N line items, the system executes 2 + 2N queries (1 order + 1 line items list + N item lookups + N inventory lookups). For a 10-item order, that's **22 SELECT queries**
- Each line item triggers two separate SELECT queries: one 3-table join for item details and one for current inventory quantity
- The order ownership check (`account.username == order.username`) happens **after** all queries have already executed, meaning unauthorized access attempts still incur full DB cost
- The `accountBean` is retrieved from session under key `"accountBean"` (not `/actions/Account.action`), which is a different key than `listOrders` uses -- this is a potential source of bugs

#### Estimated User Traffic Weight
**1-2%** of total traffic

---

### Scenario 13: User Sign-Out

- **Criticality**: **Low**
- **Category**: High Frequency User Access
- **Type**: Session-only (no DB operations)

#### User Flow Description
1. User clicks "Sign Out" link
2. HTTP session is invalidated
3. User is redirected to main catalog page

#### Endpoints Involved
| Step | URL | Handler Method |
|---|---|---|
| 1 | `GET /actions/Account.action?signoff=` | `AccountActionBean.signoff()` |
| 2 | `REDIRECT /actions/Catalog.action` | `CatalogActionBean.viewMain()` |

#### Service Methods Called
- **None**

#### Database Operations
- **None**

#### Performance Risk Factors
- `session.invalidate()` destroys the HTTP session including all session-scoped ActionBeans (CatalogActionBean, CartActionBean, OrderActionBean, AccountActionBean) and their accumulated state
- Under high load, mass session invalidation can cause garbage collection spikes if sessions held large cart or order data
- Redirect after invalidation creates a new session immediately on the next request

#### Estimated User Traffic Weight
**3-4%** of total traffic (mirrors sign-in rate)

---

## 4. Handler Method Coverage Matrix

This matrix confirms every public handler method in each ActionBean is accounted for in at least one scenario.

### CatalogActionBean

| Handler Method | Return Type | Scenario(s) | Covered? |
|---|---|---|---|
| `viewMain()` | `ForwardResolution` | Scenario 1 (Homepage) | Yes |
| `viewCategory()` | `ForwardResolution` | Scenario 2 (Browse) | Yes |
| `viewProduct()` | `ForwardResolution` | Scenario 2 (Browse) | Yes |
| `viewItem()` | `ForwardResolution` | Scenario 2 (Browse) | Yes |
| `searchProducts()` | `ForwardResolution` | Scenario 3 (Search) | Yes |
| `clear()` | `void` | Internal utility (called by other beans) | N/A (not a handler) |

### AccountActionBean

| Handler Method | Return Type | Scenario(s) | Covered? |
|---|---|---|---|
| `signonForm()` | `Resolution` | Scenario 4 (Sign-In) | Yes |
| `signon()` | `Resolution` | Scenario 4 (Sign-In), Scenario 10 (Purchase) | Yes |
| `signoff()` | `Resolution` | Scenario 13 (Sign-Out) | Yes |
| `newAccountForm()` | `Resolution` | Scenario 5 (Registration) | Yes |
| `newAccount()` | `Resolution` | Scenario 5 (Registration) | Yes |
| `editAccountForm()` | `Resolution` | Scenario 6 (Update Profile) | Yes |
| `editAccount()` | `Resolution` | Scenario 6 (Update Profile) | Yes |
| `isAuthenticated()` | `boolean` | Used by OrderActionBean and JSPs | N/A (not a handler) |
| `clear()` | `void` | Internal utility | N/A (not a handler) |

### CartActionBean

| Handler Method | Return Type | Scenario(s) | Covered? |
|---|---|---|---|
| `addItemToCart()` | `Resolution` | Scenario 7 (Add to Cart), Scenario 10 (Purchase) | Yes |
| `removeItemFromCart()` | `Resolution` | Scenario 9 (Remove from Cart) | Yes |
| `updateCartQuantities()` | `Resolution` | Scenario 8 (Update Cart) | Yes |
| `viewCart()` | `ForwardResolution` | Scenario 7 (View Cart), Scenario 10 (Purchase) | Yes |
| `checkOut()` | `ForwardResolution` | Scenario 10 (Purchase) | Yes |
| `clear()` | `void` | Called after order placement | N/A (not a handler) |

### OrderActionBean

| Handler Method | Return Type | Scenario(s) | Covered? |
|---|---|---|---|
| `listOrders()` | `Resolution` | Scenario 11 (Order History) | Yes |
| `newOrderForm()` | `Resolution` | Scenario 10 (Purchase) | Yes |
| `newOrder()` | `Resolution` | Scenario 10 (Purchase) -- multi-step | Yes |
| `viewOrder()` | `Resolution` | Scenario 12 (Order Details) | Yes |
| `clear()` | `void` | Internal utility | N/A (not a handler) |

**Result: All public handler methods are accounted for.**

---

## 5. Complete SQL Operations Inventory

Cross-reference of all MyBatis mapper XML operations with the scenarios that trigger them.

### CategoryMapper.xml

| Operation ID | Type | Table(s) | Cache | Triggered By |
|---|---|---|---|---|
| `getCategory` | SELECT | CATEGORY | Yes | Scenario 2 (viewCategory) |
| `getCategoryList` | SELECT | CATEGORY | Yes | Not directly called by ActionBeans (available in service) |

### ProductMapper.xml

| Operation ID | Type | Table(s) | Cache | Triggered By |
|---|---|---|---|---|
| `getProduct` | SELECT | PRODUCT | Yes | Scenario 2 (viewProduct) |
| `getProductListByCategory` | SELECT | PRODUCT | Yes | Scenario 2 (viewCategory), Scenario 4/5/6 (after auth) |
| `searchProductList` | SELECT | PRODUCT | Yes | Scenario 3 (searchProducts) |

### ItemMapper.xml

| Operation ID | Type | Table(s) | Cache | Triggered By |
|---|---|---|---|---|
| `getItemListByProduct` | SELECT | ITEM, PRODUCT | Yes | Scenario 2 (viewProduct) |
| `getItem` | SELECT | ITEM, INVENTORY, PRODUCT | Yes | Scenario 7 (addItemToCart), Scenario 12 (viewOrder) |
| `getInventoryQuantity` | SELECT | INVENTORY | Yes | Scenario 7 (addItemToCart), Scenario 12 (viewOrder) |
| `updateInventoryQuantity` | UPDATE | INVENTORY | Flushes cache | Scenario 10 (insertOrder) |

### AccountMapper.xml

| Operation ID | Type | Table(s) | Cache | Triggered By |
|---|---|---|---|---|
| `getAccountByUsername` | SELECT | ACCOUNT, PROFILE, SIGNON, BANNERDATA | Yes | Scenario 5/6 (after insert/update) |
| `getAccountByUsernameAndPassword` | SELECT | ACCOUNT, PROFILE, SIGNON, BANNERDATA | Yes | Scenario 4 (signon) |
| `insertAccount` | INSERT | ACCOUNT | Flushes cache | Scenario 5 (registration) |
| `insertProfile` | INSERT | PROFILE | Flushes cache | Scenario 5 (registration) |
| `insertSignon` | INSERT | SIGNON | Flushes cache | Scenario 5 (registration) |
| `updateAccount` | UPDATE | ACCOUNT | Flushes cache | Scenario 6 (update profile) |
| `updateProfile` | UPDATE | PROFILE | Flushes cache | Scenario 6 (update profile) |
| `updateSignon` | UPDATE | SIGNON | Flushes cache | Scenario 6 (update profile, conditional) |

### OrderMapper.xml

| Operation ID | Type | Table(s) | Cache | Triggered By |
|---|---|---|---|---|
| `getOrder` | SELECT | ORDERS, ORDERSTATUS | Yes | Scenario 12 (viewOrder) |
| `getOrdersByUsername` | SELECT | ORDERS, ORDERSTATUS | Yes | Scenario 11 (listOrders) |
| `insertOrder` | INSERT | ORDERS | Flushes cache | Scenario 10 (purchase) |
| `insertOrderStatus` | INSERT | ORDERSTATUS | Flushes cache | Scenario 10 (purchase) |

### LineItemMapper.xml

| Operation ID | Type | Table(s) | Cache | Triggered By |
|---|---|---|---|---|
| `getLineItemsByOrderId` | SELECT | LINEITEM | Yes | Scenario 12 (viewOrder) |
| `insertLineItem` | INSERT | LINEITEM | Flushes cache | Scenario 10 (purchase) |

### SequenceMapper.xml

| Operation ID | Type | Table(s) | Cache | Triggered By |
|---|---|---|---|---|
| `getSequence` | SELECT | SEQUENCE | Yes | Scenario 10 (purchase - ID gen) |
| `updateSequence` | UPDATE | SEQUENCE | Flushes cache | Scenario 10 (purchase - ID gen) |

**Result: All 22 mapper operations are accounted for in the scenarios above.**

---

## 6. Transactional Behavior Analysis

The following service methods are annotated with `@Transactional` (Spring declarative transaction management):

| Service | Method | Operations in Transaction | Risk Level |
|---|---|---|---|
| `AccountService` | `insertAccount()` | 3 INSERTs (ACCOUNT + PROFILE + SIGNON) | Medium -- locks 3 tables briefly |
| `AccountService` | `updateAccount()` | 2-3 UPDATEs (ACCOUNT + PROFILE + optionally SIGNON) | Medium -- row-level locks |
| `OrderService` | `insertOrder()` | 1 SELECT + 1 UPDATE (SEQUENCE) + N UPDATEs (INVENTORY) + 1 INSERT (ORDERS) + 1 INSERT (ORDERSTATUS) + N INSERTs (LINEITEM) | **Critical** -- longest transaction, most locks |
| `OrderService` | `getOrder()` | Multiple SELECTs (read-only but transactional for consistency) | Low -- read locks only |

### Key Transactional Risk: `OrderService.insertOrder()`

This is the most performance-critical transactional method. Under concurrent load:

1. **Sequence contention**: The `getNextId("ordernum")` method does SELECT + UPDATE on a single row, creating a serial bottleneck
2. **Inventory lock escalation**: Multiple concurrent orders for the same item will contend for the same INVENTORY row locks
3. **Transaction duration**: For an order with N items, the transaction performs 4 + 2N operations, meaning a 5-item order executes 14 SQL statements within a single transaction
4. **Rollback impact**: If any operation fails (e.g., inventory update), all operations roll back, including any inventory decrements already applied to other items

---

## 7. MyBatis Cache Configuration

All mapper XML files use `<cache />` (MyBatis second-level cache with default settings):

| Mapper | Cache Enabled | Impact |
|---|---|---|
| CategoryMapper | Yes | Low-risk: Category data is static |
| ProductMapper | Yes | Low-risk: Product data is mostly static |
| ItemMapper | Yes | **Medium-risk**: Inventory quantities are cached; `updateInventoryQuantity` flushes the entire ItemMapper cache, but stale reads are possible between cache flush and next query |
| AccountMapper | Yes | Low-risk: Account data changes infrequently |
| OrderMapper | Yes | Medium-risk: New orders flush cache; frequent order placement invalidates cache often |
| LineItemMapper | Yes | Medium-risk: Same as OrderMapper |
| SequenceMapper | Yes | **High-risk**: Sequence values are cached but immediately invalidated by updates; the cache provides minimal benefit here and adds overhead |

### Cache Concern for Load Testing
Under concurrent load, MyBatis second-level cache can cause:
- **Stale inventory reads**: After one user purchases an item (flushing ItemMapper cache), another user's `isItemInStock()` call may use a stale cached value before the cache is fully invalidated
- **Cache thrashing**: Frequent writes (orders, registrations) flush the entire mapper namespace cache, reducing cache hit rates for read-heavy operations
- **Memory pressure**: All cached objects are stored in memory; under high load with many products/orders, cache size may grow significantly

---

## 8. Summary Table

| # | Scenario | Criticality | Type | DB Operations | Transactional? | Traffic Weight | Key Risk |
|---|---|---|---|---|---|---|---|
| 1 | Homepage / Main Catalog View | High | Read | None | No | 25-30% | Session creation overhead |
| 2 | Browse Category -> Product -> Item | High | Read | 5 SELECTs (incl. 3-table join) | No | 30-35% | Multi-table joins, sequential queries |
| 3 | Search Products | Medium | Read | N SELECTs (one per keyword) | No | 5-8% | LIKE with leading wildcard, no index, multi-query for multi-word |
| 4 | User Sign-In | High | Read | 2 SELECTs (incl. 4-table join) | No | 8-10% | 4-table join for auth, gateway bottleneck |
| 5 | User Registration | Medium | Write | 3 INSERTs + 2 SELECTs | **Yes** | 1-2% | Transactional 3-table insert, PK constraints |
| 6 | Update User Profile | Medium | Mixed | 2-3 UPDATEs + 2 SELECTs | **Yes** | 1-2% | Transactional multi-table update |
| 7 | Add Item to Cart + View Cart | High | Read | 0-2 SELECTs | No | 10-12% | Stock check + 3-table join per new item |
| 8 | Update Cart Quantities | Low | None | None | No | 2-3% | Session-only, minimal risk |
| 9 | Remove Item from Cart | Low | None | None | No | 1-2% | Session-only, minimal risk |
| **10** | **Full Purchase Flow (E2E)** | **Critical** | **Mixed** | **4 + 2N ops (SELECTs + UPDATEs + INSERTs)** | **Yes** | **3-5%** | **Sequence bottleneck, inventory locks, long transaction, N+1 writes** |
| 11 | View Order History | Medium | Read | 1 SELECT (2-table join) | No | 2-3% | No pagination, unbounded result set |
| 12 | View Order Details | Medium | Read | 2 + 2N SELECTs | **Yes** (read) | 1-2% | N+1 query problem, 3-table joins per line item |
| 13 | User Sign-Out | Low | None | None | No | 3-4% | Session invalidation GC impact |

### Priority Ranking for Load Test Implementation

| Priority | Scenarios | Rationale |
|---|---|---|
| **P0 (Must Test)** | Scenario 10 (Full Purchase), Scenario 2 (Browse), Scenario 4 (Sign-In) | Highest business impact + highest risk DB operations |
| **P1 (Should Test)** | Scenario 1 (Homepage), Scenario 7 (Add to Cart), Scenario 3 (Search) | High traffic + DB read load |
| **P2 (Nice to Test)** | Scenario 11 (Order History), Scenario 12 (Order Details), Scenario 5 (Registration), Scenario 6 (Update Profile) | Medium traffic, specific risk patterns |
| **P3 (Low Priority)** | Scenario 8 (Update Cart), Scenario 9 (Remove from Cart), Scenario 13 (Sign-Out) | Session-only, minimal DB impact |

### Read vs. Write Classification

| Classification | Scenarios | Total Traffic Weight |
|---|---|---|
| **Read-Heavy** | 1, 2, 3, 4, 7, 11, 12 | ~80-85% |
| **Write-Heavy** | 5, 10 | ~4-7% |
| **Mixed (Read+Write)** | 6 | ~1-2% |
| **No DB** | 8, 9, 13 | ~6-9% |
