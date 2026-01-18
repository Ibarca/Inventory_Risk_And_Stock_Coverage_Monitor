# Inventory Projection Dashboard – BigQuery SQL

During my experience working across multiple category management teams, I repeatedly observed a lack of reliable, forward-looking inventory visibility at SKU level. This often resulted in avoidable stockouts caused not by missing data, but by the absence of a structured projection view. This project represents the dashboard I always wished my teams had access to and is therefore strongly grounded in real operational use cases.

This repository contains a production-ready BigQuery SQL query that powers an end-to-end Inventory Projection Dashboard. The query integrates historical sales, inventory snapshots, incoming purchase orders, and calendar logic into a single analytical dataset designed for inventory risk assessment, service-level forecasting, and SKU prioritization.

The dashboard is built for Category Management, Supply Chain, and Operations teams, enabling scalable, weekly, SKU-level decision making based on consistent and transparent data logic.

<img width="1284" height="709" alt="Screenshot 2026-01-16 at 12 19 59" src="https://github.com/user-attachments/assets/719e66c6-4315-49d1-bb89-27cb622f8d9c" />

---

## 📌 What this project does

The query produces a weekly, ISO-calendar inventory projection at SKU level, applying a true stock floor that never allows inventory to drop below zero. The results are enriched with multiple analytical layers designed to support proactive inventory management:

• Projected inventory units and value over time: Enabling early detection of upcoming stockouts and timely corrective actions, curcial for the manager to preventibly react on time

• Inbound inventory visibility and valuation: Providing an immediate overview of open purchase orders per SKU, both in units and monetary value

• Demand projections based on historical sales velocity: Demand is derived from average sales velocities from the previous year; this logic can easily be replaced by more advanced forecasting techniques, which are explored in a separate project

• Projected service level over a forward-looking horizon: The dashboard displays the expected service level over the next six months based on projected inventory availability. Since last orders are expected to arrive between 6 and 7 months from the initial date, 6 months is considered a reasonable time horizon to measure service.

• First expected stockout date and reach in weeks: Allowing management to assess the severity and urgency of potential stockouts and respond in time

• ABC classification based on revenue contribution: SKUs are ranked by prior-year revenue and segmented using Pareto-based ABC classification to support effective prioritization. The Paretto logic used is 40-40-20

• Stockout risk and sales velocity segmentation: SKUs are grouped into clear risk and velocity buckets, making it easy to identify items at stockout risk as well as dead or slow-moving inventory

All transformations and calculations are executed in BigQuery SQL, producing a dataset that is directly ready for consumption by a BI layer. Metric names are standardized and simplified in the visualization layer, while charts and tables are arranged following core dashboard design principles to ensure clarity and usability.

The resulting dataset is consumed directly by Looker Studio, which render the inventory projection dashboard shown above.

---

## 🧠 Business use cases

• Identify SKUs at risk of stockout weeks in advance

• Prioritize replenishment decisions using ABC logic

• Monitor projected service levels at SKU and category level

• Evaluate the impact of incoming purchase orders on future availability

• Track inventory units and value evolution over time

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

<img width="1536" height="1024" alt="Inventory projection data flow diagram" src="https://github.com/user-attachments/assets/851fa15a-58e7-46b2-a48e-2ddc7589a442" />


---


## ⚙️ Technical approach

• Recursive CTEs are used to model true week-over-week inventory evolution

Inventory is a stateful metric: the stock you have in week t depends on the stock remaining in week t–1. That dependency is why the query uses a recursive CTE (weekly_projection).

The inventory projection follows a stateful, week-by-week logic that mirrors how stock evolves in real operations. The model starts with an anchor week, which represents the first week of the projection horizon. For this initial step, inventory is calculated using the starting stock snapshot, adjusted by adding incoming quantities for the week and subtracting expected demand. A strict floor is applied to ensure inventory never drops below zero, reflecting the physical constraint of real stock.

From that point onward, the projection advances recursively. Each subsequent week builds directly on the projected inventory level of the previous week, again incorporating inbound deliveries and expected demand for the current week. By continuously carrying forward the inventory state, the model accurately captures the cumulative impact of demand and replenishment decisions over time.

This recursive approach avoids common pitfalls such as treating each week as an isolated calculation or allowing negative stock levels, resulting in a realistic and operationally sound inventory trajectory.

![F465A7F4-20AC-4429-ACEF-43D01E9C9DE6](https://github.com/user-attachments/assets/f9d08216-bc8b-4cc8-b8cd-94f92d42099f)


• Window functions support ranking, service level calculation, and reach logic

Window functions are the core analytical layer of this query, enabling advanced KPIs while preserving SKU × week granularity.

They are used first for ranking and prioritization, assigning revenue-based ranks to SKUs and calculating cumulative revenue contribution to support ABC (Pareto) classification. This allows large assortments to be quickly prioritized based on business impact.

<img width="612" height="582" alt="image" src="https://github.com/user-attachments/assets/b59795a2-ee0e-40b2-9c63-c0c6002c25a0" />


They also power the projected service level, calculated as a forward-looking availability ratio over the next 24 weeks. By averaging a binary in-stock indicator across a future window, the model estimates how reliably each SKU will be available over time.

Finally, window functions support the reach and stockout logic by identifying the first projected stockout week and date, counting total projected weeks, and deriving the number of weeks until stockout. Together, these metrics provide a clear, actionable view of urgency and inventory risk.

<img width="845" height="349" alt="image" src="https://github.com/user-attachments/assets/c874d602-b0c2-45e3-b51d-845e8fd56ea2" />




• ISO calendar logic ensures consistent weekly alignment

ISO calendar logic ensures that demand, inbound supply, and inventory are all evaluated on the same consistent weekly timeline. By standardizing everything to ISO weeks starting on Monday, the model avoids year-transition edge cases and misaligned weeks. A complete ISO-week calendar is generated in SQL and cross-joined with SKUs to guarantee continuous weekly rows, even when no activity exists, so the recursive projection never breaks. Inbound purchase orders are aggregated using the same ISO-week definition, ensuring that supply, demand, and inventory movements align perfectly.

By enforcing a single, consistent ISO-week structure across all inputs, the model avoids misalignment between sales, orders, and stock movements, producing a reliable and operationally meaningful weekly inventory projection.

<img width="565" height="221" alt="image" src="https://github.com/user-attachments/assets/a98c6d45-430f-4895-a9b3-443443e3e9ff" />


• All heavy transformations run in SQL to avoid BI-layer complexity

All heavy transformations are executed directly in SQL to leverage BigQuery’s scalability and keep the BI layer simple and performant. Complex logic such as demand aggregation, ISO-week alignment, recursive inventory projection, and KPI calculations is handled upstream, producing a clean, fully enriched dataset. This allows Looker Studio to focus purely on visualization and interaction, avoids duplicated logic across dashboards, ensures consistent KPI definitions, and makes the overall solution easier to scale, maintain, and audit.

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

