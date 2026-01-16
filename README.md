# Inventory Projection Dashboard – BigQuery SQL

During my experience working across multiple category management teams, I repeatedly observed a lack of reliable, forward-looking inventory visibility at SKU level. This often resulted in avoidable stockouts caused not by missing data, but by the absence of a structured projection view. This project represents the dashboard I always wished my teams had access to and is therefore strongly grounded in real operational use cases.

This repository contains a production-ready BigQuery SQL query that powers an end-to-end Inventory Projection Dashboard. The query integrates historical sales, inventory snapshots, incoming purchase orders, and calendar logic into a single analytical dataset designed for inventory risk assessment, service-level forecasting, and SKU prioritization.

The dashboard is built for Category Management, Supply Chain, and Operations teams, enabling scalable, weekly, SKU-level decision making based on consistent and transparent data logic.

<img width="1284" height="709" alt="Screenshot 2026-01-16 at 12 19 59" src="https://github.com/user-attachments/assets/719e66c6-4315-49d1-bb89-27cb622f8d9c" />

---

## 📌 What this project does

The query produces a weekly, ISO-calendar inventory projection at SKU level, applying a true stock floor that never allows inventory to drop below zero. The results are enriched with multiple analytical layers designed to support proactive inventory management:

• Projected inventory units and value over time – Enabling early detection of upcoming stockouts and timely corrective actions

• Inbound inventory visibility and valuation – Providing an immediate overview of open purchase orders per SKU, both in units and monetary value

• Demand projections based on historical sales velocity – Demand is derived from average sales velocities from the previous year; this logic can easily be replaced by more advanced forecasting techniques, which are explored in a separate project

• Projected service level over a forward-looking horizon – The dashboard displays the expected service level over the next six months based on projected inventory availability

• First expected stockout date and reach in weeks – Allowing management to assess the severity and urgency of potential stockouts and respond in time

• ABC classification based on revenue contribution – SKUs are ranked by prior-year revenue and segmented using Pareto-based ABC classification to support effective prioritization

• Stockout risk and sales velocity segmentation – SKUs are grouped into clear risk and velocity buckets, making it easy to identify items at stockout risk as well as dead or slow-moving inventory

All transformations and calculations are executed in BigQuery SQL, producing a dataset that is directly ready for consumption by a BI layer. Metric names are standardized and simplified in the visualization layer, while charts and tables are arranged following core dashboard design principles to ensure clarity and usability.

The resulting dataset is consumed directly by Looker Studio, which render the inventory projection dashboard shown above.

---

## 🧠 Business use cases

• Identify SKUs at risk of stockout weeks in advance

• Prioritize replenishment decisions using ABC logic

• Monitor projected service levels at SKU and category level

• Evaluate the impact of incoming purchase orders on future availability

• Track inventory value evolution over time

• Segment assortment by sales velocity and risk


This logic has been implemented and refined in real business environments and is designed to handle **thousands of SKUs efficiently**.

---

## 🏗️ Data sources

All data used in this project is simulated to closely reflect real-world operating conditions. Incoming purchase orders include delivery deviations of up to ±10 days, depending on the supplier, to mirror typical lead-time variability. In addition, selected purchase orders were intentionally removed to create controlled stockout scenarios, allowing the dashboard to demonstrate different levels of inventory risk and stress-test the projection logic.

The initial goal was to build a daily inventory projection. However, at SKU scale this approach proved too resource-intensive in BigQuery. To balance realism and performance, the model was redesigned around the standardized ISO week calendar, resulting in a 53-week projection horizon that aligns well with operational planning cycles.

The query integrates the following datasets:

• SKU master data (category, brand, supplier, unit cost)
• Historical daily sales data (used for demand and revenue estimation)
• Inventory snapshot at the start of the projection period
• Incoming purchase orders with expected delivery dates
• ISO-week calendar generated directly in SQL for the projection horizon

![Data_sources](https://github.com/user-attachments/assets/4ef4e223-1a91-4783-906f-9d6234a7854e)


⚙️ Technical approach

• Recursive CTEs are used to model true week-over-week inventory evolution

Inventory is a stateful metric: the stock you have in week t depends on the stock remaining in week t–1. That dependency is why the query uses a recursive CTE (weekly_projection).

In my logic:
• Anchor week (rn = 1): starts from the initial stock snapshot (inventory_snapshot_2026_01_01) plus inbound for that week minus demand for that week, with a true floor at 0 using GREATEST(0, …).
• Recursive step (rn = previous rn + 1): each next week uses the previous week’s inventory_projection as the base, then applies weekly inbound and weekly demand again.

This prevents common mistakes like treating each week independently or allowing negative stock, and it produces a realistic inventory trajectory that behaves like real operations.

![F465A7F4-20AC-4429-ACEF-43D01E9C9DE6](https://github.com/user-attachments/assets/f9d08216-bc8b-4cc8-b8cd-94f92d42099f)


• Window functions support ranking, service level calculation, and reach logic
• ISO calendar logic ensures consistent weekly alignment
• All heavy transformations run in SQL to avoid BI-layer complexity

This approach keeps business logic centralized, version-controlled, and easy to audit.

📈 Output & BI integration

The final output is a BI-ready table with standardized metric names and clear business labels. It can be directly connected to tools such as:

• Looker Studio
• Google Sheets
• Other BI or reporting layers

The dashboard built on top of this dataset follows core dashboard design principles, combining KPIs, tables, and time-series views into an intuitive layout for operational decision making.

🗂️ Repository structure

/sql
inventory_projection.sql

/docs
dashboard_screenshot.png

README.md

🧪 Data note

The data used in this repository may be synthetic or anonymized. The goal of this project is to demonstrate analytical design, SQL quality, and real-world inventory logic, not to expose proprietary information.

👥 Intended audience

• Category Managers
• Supply Chain and Inventory Planners
• BI and Data Analytics teams
• Hiring managers reviewing analytics portfolios

This project represents a realistic, production-style inventory analytics use case, suitable for both operational environments and professional data portfolios.

