# 📊 Power BI DAX Cookbook

> A practical collection of DAX patterns, dimensional modeling techniques, and performance tips I've used building enterprise Power BI dashboards — most recently on Microsoft Fabric Warehouse / Lakehouse architectures.

---

## 🧭 Philosophy

Most DAX you'll find online is either trivial (`SUM(Sales[Amount])`) or so abstract you can't actually use it. This cookbook focuses on **real patterns for real problems** that come up in enterprise reporting:

- Time intelligence that doesn't break with custom fiscal calendars
- Measures that perform on tables with 10M+ rows
- Row-level security patterns that actually scale
- Models that BI consumers can self-serve without breaking

---

## 📖 What's inside

### 🕰️ Time intelligence patterns

#### Year-to-date with fiscal year override

The built-in `TOTALYTD` assumes calendar year. Most real businesses don't operate on calendar year. Here's a version that respects a custom fiscal year start (parameterized via a `Date` dimension with `FiscalYear` and `FiscalQuarter` columns):

```dax
Sales YTD Fiscal :=
CALCULATE(
    [Total Sales],
    FILTER(
        ALL( 'Date' ),
        'Date'[FiscalYear] = MAX( 'Date'[FiscalYear] )
            && 'Date'[Date] <= MAX( 'Date'[Date] )
    )
)
```

#### Same period prior year (handles leap years)

```dax
Sales SPLY :=
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR( 'Date'[Date] )
)

Sales YoY Growth :=
DIVIDE(
    [Total Sales] - [Sales SPLY],
    [Sales SPLY]
)
```

**Pro tip:** always wrap year-over-year measures in `DIVIDE()` not `/`. It handles divide-by-zero implicitly and returns `BLANK()` instead of an error.

---

### 🧮 Filter context manipulation

#### Calculate measure ignoring slicer filters

A common ask: "I want to see this product's sales as a % of total, regardless of what's filtered."

```dax
Sales % of Total :=
DIVIDE(
    [Total Sales],
    CALCULATE( [Total Sales], ALL( 'Product' ) )
)
```

The `ALL( 'Product' )` removes any filter from the Product table — so the denominator is "total sales across all products, with all other filters still applied."

#### Top N with "Others" bucket

Showing top 10 products + lumping the rest as "Others":

```dax
Sales Top 10 :=
VAR _CurrentProductRank =
    RANKX(
        ALL( 'Product'[ProductName] ),
        [Total Sales],
        ,
        DESC
    )
RETURN
    IF( _CurrentProductRank <= 10, [Total Sales], BLANK() )

Sales Others :=
VAR _Top10Total =
    SUMX(
        TOPN( 10, ALL( 'Product'[ProductName] ), [Total Sales] ),
        [Total Sales]
    )
RETURN
    CALCULATE( [Total Sales], ALL( 'Product' ) ) - _Top10Total
```

---

### 🏗️ Dimensional modeling patterns

#### The star schema is not optional

90% of "Power BI is slow" complaints I've seen trace back to bad models — usually a single flat table or a snowflake when it should be a star.

**Rules I follow:**

1. **One fact table per business process** (sales, inventory movements, financial entries). Don't smash multiple processes into one fact.
2. **All filtering goes through dimensions**, never directly on fact. Slicers, filter panes, all of it.
3. **Surrogate keys on dimensions** — not the business key. The business key changes; the surrogate doesn't.
4. **Type 2 SCD on dimensions where history matters** (customers, products). Type 1 for the rest.
5. **Date dimension is mandatory.** Not optional. Even if your fact only has one date, make a dim_date and relate to it.

#### Conformed dimensions across multiple facts

When you have multiple fact tables (sales, returns, inventory), they should share dimensions — not duplicate them. Example: `dim_product` is used by `fact_sales`, `fact_returns`, AND `fact_inventory_snapshot`. One dimension, three relationships, three fact tables.

This enables cross-process analysis: "What's our gross margin (sales fact) on products that have high return rates (returns fact)?" — one query, two facts, conformed via dim_product.

---

### ⚡ Performance patterns

#### Variables ≫ repeated CALCULATE

Don't do this:

```dax
Bad Measure :=
IF(
    CALCULATE( SUM( Sales[Amount] ), Region[Name] = "LATAM" ) > 10000,
    CALCULATE( SUM( Sales[Amount] ), Region[Name] = "LATAM" ) * 1.1,
    CALCULATE( SUM( Sales[Amount] ), Region[Name] = "LATAM" )
)
```

That's three identical context transitions. Do this:

```dax
Good Measure :=
VAR _LatamSales = CALCULATE( SUM( Sales[Amount] ), Region[Name] = "LATAM" )
RETURN
    IF( _LatamSales > 10000, _LatamSales * 1.1, _LatamSales )
```

One calculation, reused. Massive difference at scale.

#### Avoid bidirectional relationships unless you really need them

Bidirectional relationships seem helpful, but they:
- Make the model harder to reason about
- Often cause unexpected filter propagation
- Can multiply query times

Default to single-direction. Use `CROSSFILTER()` in specific measures if you need bidirectional behavior for one calculation.

#### Pre-aggregate when you can

For large fact tables (10M+ rows), build a pre-aggregated table at daily/weekly grain and configure aggregations in Power BI. The engine picks the lowest-granularity match — visuals showing monthly totals query the aggregation, not the detail.

---

### 🔒 Row-level security at scale

Static RLS (single role per user) works for small teams. For larger organizations, dynamic RLS using `USERPRINCIPALNAME()`:

```dax
[Salesperson Filter] =
'Salesperson'[Email] = USERPRINCIPALNAME()
```

Combined with a `dim_user_permission` table (mapping users to allowed regions / customers / products), this scales to thousands of users without role explosion.

---

### 🧪 Microsoft Fabric specifics

If you're using Fabric (Warehouse / Lakehouse), some patterns shift:

- **Direct Lake mode** beats Import for huge datasets — but it doesn't support all DAX functions. Test early.
- **Aggregations** still apply, configured at the dataset level.
- **Notebook + Spark** for prep, then expose via Direct Lake — pattern works well.
- **Copilot for Power BI** has gotten genuinely useful for natural-language Q&A; configure it on critical dashboards.

---

## 📂 Repository Structure

```
powerbi-dax-cookbook/
├── README.md                            ← you are here
├── patterns/
│   ├── 01-time-intelligence.md
│   ├── 02-filter-context.md
│   ├── 03-dimensional-modeling.md
│   ├── 04-performance.md
│   ├── 05-rls-patterns.md
│   └── 06-fabric-specifics.md
├── dax-snippets/                        ← copy-paste ready measures
│   ├── time-intelligence.dax
│   ├── ranking-and-top-n.dax
│   ├── period-comparisons.dax
│   └── financial-ratios.dax
└── examples/                            ← .pbix-style screenshots & data models
```

---

## 🤝 About me

**Roberto Iguardia** — Data Engineer building production BI on the Microsoft stack. Power BI + DAX + Microsoft Fabric specialist.

- 🌐 [alejandroiguardia.netlify.app](https://alejandroiguardia.netlify.app/)
- 💼 [LinkedIn](https://www.linkedin.com/in/alejandro-iguardia-data-engineer)
- 📧 robertalerosales12@gmail.com

If you've got a DAX problem you can't crack, open an issue — I'm happy to take a look.v
