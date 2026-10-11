## DataWarehouse Schema Configuration

# 🌟 Star and Snowflake Schema Architectures
Both of these models are built around the same three transactional fact tables (`FactBillItems`, `FactOrderItems`, `FactCartItems`), meaning they capture the exact same operational events. The big divide between them is purely how we handle the descriptive lookup data surrounding those facts.


# The Gold Star Schema

  * This approach is all about flat, wide data structures. Instead of worrying about duplicate text, everything describing a business entity is packed right into that entity's primary table.
  * For example, `DimLocations` holds full physical shipping and billing text blocks right in its own rows, and `DimUsers` keeps all profile elements, pre-flattened contact lists, and roles side-by-side with your denormalized `clinicName` strings.
  * It results in a clean, simple layout where your fact tables sit in the middle, directly connected to five single-tier dimension tables.

# The Gold Snowflake Schema


  * This model focuses on reducing repeating text strings to keep the database normalized.
  * To do this, it splits high-cardinality address strings out of `DimLocations`, `DimUsers`, and `DimSuppliers` and groups them into a separate `DimGeography` table.
  * It also strips the general ledger accounting codes out of the order lines and isolates them inside a dedicated `DimAccountCodes` lookup table.
  * However, to protect retrieval speeds, `clinicName` is kept denormalized inside the `DimUsers` table rather than being forced out into a separate sub-table.
  * It creates a branching web where you have to hop through secondary tables to get the full picture.



## 🏆 The Choice: Why the Gold Star Schema is the Best Fit

If I have to choose one to run a production environment on Azure Databricks and PostgreSQL, I would choose the Gold Star Schema. Normalizing dimensions to save a few megabytes of disk space is an outdated practice that introduces unnecessary complexity.

**It Eliminates Slow, Multi-Hop Joins**
  * Reporting tools like Power BI and machine learning feature pipelines need to fetch data fast. In a Star Schema, if a model wants to calculate "Total Spend by Clinic City," it only needs a single, direct join between `FactBillItems` and `DimUsers`.
  * In a Snowflake model, the database engine has to jump from the Fact table to the User table, and then run a secondary join over to the Geography table.
  * When you are processing millions of rows, those nested joins slow down performance.
    
**Storage is Cheap, but Compute is Expensive**
  * The main reason people used to build Snowflake schemas was to save storage space by avoiding duplicate text.
  * Today, storage in ADLS Gen2 and PostgreSQL is incredibly cheap, and modern database engines compress wide text fields automatically.
  * It doesn't make sense to sacrifice query speed and increase your cluster compute costs just to save a negligible amount of disk space.
    
**It Keeps the Code Simple and Easy to Maintain**
  * PySpark scripts are already outputting wide, clean, denormalized data at the Silver stage (`silverBillsFlat`, `silverOrdersFlat`, `silverCartsAndUsersFlat`). Loading a Star Schema is a straightforward, direct append.
  * Forcing that flat data into a Snowflake structure means writing complex, multi-layered loading loops to manage surrogate keys for sub-dimensions, track parent-child lookup states, and handle step-by-step dependency checks. That added complexity just creates more places for code to break.
    
**Instant Feature Extraction for Machine Learning**
  * Machine learning models thrive on flat, comprehensive data vectors. By keeping descriptive properties—like `clinicName`, contact lists, roles, and facility configurations—stored flat inside your dimensions, your models can pull clean features instantly at runtime without waiting for the database to calculate a massive, relational tree.


## Why we chosed this Dimension and Fact Tables
## 📐 Conformed Dimension Tables

**1. DimUsers (User Master Profile)**
  * Customer account profiles do not arrive as a standalone data stream; they are hidden inside the shopping cart documents.
  * This table pulls out those embedded keys during staging to build a dedicated registry of users, pre-flattening multi-valued arrays like roles and department IDs into clean strings.
  * This gives your analytics team a central lookup to track employee purchasing habits across all transaction workflows while keeping `clinicName` flat to provide high-speed, zero-join feature sets for your machine learning models.
    
**2. DimLocations (Physical Sites & Facilities)**
  * Geographic data and drop-off facility details are repeated continuously across incoming bills and purchase orders.
  * This table standardizes those uneven, free-text address strings into a structured registry of physical business sites.
  * It enables cross-functional regional spend mapping, allowing business intelligence tools to instantly compare total ordered spend against final billed costs for any specific clinic branch, accounting warehouse, or office site.

**3. DimSuppliers (Procurement Distribution Partners)**
  * Your supply chain interacts with multiple major distribution partners whose details appear across entirely different document types.
  * This table strips out redundant vendor strings from incoming transactions and captures a single, unified business identity for each procurement entity.
  * It acts as the master ledger to audit vendor fulfillment performance, track historical price trends, and manage volume-based contract negotiations across all workflows.
    
**4. DimProducts (Unified Catalog Ledger)**
  * Specific product details, images, and backorder flags are embedded deep within line item grids and order arrays.
  * This table isolates those descriptive product properties away from volatile transaction rows into an organized master index.
  * It establishes a clean catalog repository to analyze product performance, track item lifecycle shifts, detect description differences across distinct vendors, and flag supply chain constraints by tracking backorder frequencies over time.
    
**5. DimDate (Master Temporal Timeline)**
  * Invoices, purchase orders, and web-crawler price checks operate on completely separate processing schedules.
  * This table maps those independent, raw timestamps to a standardized, unified calendar axis, breaking them down into discrete components like years, months, and quarters.
  * This calendar anchor makes it simple to track seasonal spend patterns, run quarterly financial audits, and measure critical turnaround windows like the exact duration from order placement to final invoice processing.

## 📊 Fact Tables

**1. FactBillItems (Accounts Payable & Financial Truth)**
  * This table captures data at the lowest level of financial detail—the individual line items packaged on a supplier's invoice document (`silverBillsFlat`).
  * By breaking down bulk document costs into granular, itemized rows, it functions as your definitive financial source of truth for all ledger auditing and compliance reporting.
  * Capturing precise metrics like `lineExtendedPrice`, `documentTaxAmount`, and `documentDiscountAmount` allows you to run automated accounts payable reviews and audit finalized invoiced totals against original contract rates.
    
**2. FactOrderItems (Upstream Purchasing & Operational Friction)**
  * This table logs data at the individual item level within a procurement purchase order sequence (`silverOrdersFlat`), capturing transactional volume before the final billing or invoicing event takes place.
  * It tracks upstream purchasing behavior and operational pipeline health across multiple sub-vendor nodes.
  * Storing metrics like `orderedQuantity` alongside system execution counters (`successCount`, `errorCount`) allows you to isolate vendor communication friction points and perform automated order-to-bill lifecycle reconciliation.
    
**3. FactCartItems (Market Intelligence & Price Benchmarking)**
  * This table flattens your user shopping cart maps alongside live crawled vendor price matrices (`silverCartsAndUsersFlat`).
  * It serves as a competitive market intelligence engine to optimize supply chain costs by storing direct price matrices from multiple industry competitors side-by-side with real-time web crawler prices.
  * Having this ceiling and floor matrix data readily available provides the exact datasets needed to run cost-saving simulations, evaluate real-time market price differences, and automatically identify optimized procurement paths for your supply chain.


## Add Star and Snowflake Schema mapping and Configuration diagrams here

# Star Schema Configuration Diagram

![StarSchema](../PNG_Diagram/StarSchema_Configuration.png)

# Snowflake Schema Configuration Diagram

![SnowflakeSchema](../PNG_Diagram/SnowflakeSchema_Configuration.png)
