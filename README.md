# Insurance Dashboard (Databricks Apps, Streamlit) on a Genie space

Same lavender / navy / violet design as before, rebuilt on the Gold tables from `Synthetic_Data_To_Gold`, read through a **Genie space you attach manually**.
With no space attached the app runs on built-in demo data in the **same columns**, so you can check the UI first.

## Set up
1. In the Genie space, add these tables (they live in `synthetic.gold2` by default): `gold_customer_profile`, `gold_property_profile`, `gold_policy_report`, `gold_claim_report`
   (optional, for the disaster chart: `fact_nfip_claim`, `dim_disaster_declaration`).
2. Create the app (Compute -> Apps -> Custom), upload this folder. Under **Resources** add the Genie space with key `genie-space`
   (or set `GENIE_SPACE_ID` to the id after `/rooms/` in the Genie URL).
3. Give the app's **service principal**: CAN RUN on the Genie space, CAN USE on the space's SQL warehouse, and USE CATALOG / USE SCHEMA / SELECT on the Gold schema.
4. Deploy. First load takes about a minute (four Genie questions); results are cached for an hour and shared by all users. "Refresh data" at the bottom of the sidebar re-asks.
5. If your catalog/schema differ, change `GOLD_CATALOG` / `GOLD_SCHEMA` in `app.yaml`.

## How the data is read (and why)
- Genie allows about **5 questions per minute per workspace** and returns **at most 5,000 rows** per result. So the app asks **4 wide questions once**
  (one per Gold table), caches them, and paces requests (max 4 per minute, leaving one for the "Ask Genie" page). Charts are computed in the app, not per-chart Genie calls.
- Each question names the table and its columns and asks for raw rows. On-screen source details were removed on purpose; the error screen and "Run diagnostics" still show the SQL when something fails. If Genie returns a different shape the app
  stops with a clear message (missing columns, text-only answer). If a table has more than 5,000 rows the app warns that the result is truncated.
- **Ask Genie** page: free-form questions with the SQL and result table.

## Which Gold table feeds which step
| Step | Source |
|---|---|
| 1 Customer | gold_customer_profile |
| 2-4 Property, location, exposure & risk | gold_property_profile |
| 5 Quote, 6 Underwriting decision, 8 Endorsements | **not in Gold.** Shown as "not in Gold" with the nearest honest proxy (pricing and risk on file) |
| 7 Policy bind | gold_property_profile + gold_policy_report |
| 9 Claim | gold_claim_report + gold_property_profile (date of loss only, no FNOL / reserve / close dates) |
| 10 Loss history, 11 Mitigation | gold_property_profile |
| 12 Renewal | gold_customer_profile / gold_property_profile (due dates, pending flag) |

## Run locally (demo data)
`pip install -r requirements.txt && streamlit run app.py`

## Troubleshooting: "PERMISSION_DENIED ... No access to 'synthetic.gold2.*'"
Unity Catalog permission, not code. The app runs as its **service principal**; it needs (run as the table owner or an admin; the principal is the app's application id,
shown on Compute -> Apps -> your app -> Authorization, and printed on the app's error screen):
```sql
GRANT USE CATALOG ON CATALOG synthetic TO `<application-id>`;
GRANT USE SCHEMA, SELECT ON SCHEMA synthetic.gold2 TO `<application-id>`;
```
plus CAN RUN on the Genie space and CAN USE on its SQL warehouse. Schema-level SELECT covers every table in the space, including the dim_/fact_ tables.

## Troubleshooting: "Genie request failed ... MessageStatus.FAILED"
The app now prints Genie's own reason (for example `TABLES_MISSING_EXCEPTION`). Then:
1. Open the Genie space in Databricks and paste: `Show all rows and all columns from synthetic.gold2.gold_customer_profile. No filter and no limit.` If that fails there too, the problem is in the space (tables not added, wrong catalog/schema name, warehouse stopped), not the app.
2. If it works in the space UI but not in the app, it is the app's service principal: CAN RUN on the space, CAN USE on the warehouse, USE CATALOG / USE SCHEMA / SELECT on the Gold schema.
3. On the error screen click **Run diagnostics** (one tiny question per table) to see which table fails, or **Use demo data for now** to keep working.
4. If Genie adds a `LIMIT` to the SQL the app warns that the table may be incomplete. Add a space instruction such as "When asked for all rows, do not add a LIMIT."

## Navigation
Sidebar: Dashboard (Overview, Process Flow, Customer 360, Portfolio & Geography, Ask Genie) and Records (Customers, Properties, Policies, Claims, Renewals).
Every Overview tile and card opens its list (Properties tile -> property list, Open claims -> open claims, Renewals -> renewals due in 60 days, ...).
Lists have search, filters, CSV export and "select a row to open that customer's journey". The header search box searches customers, properties, policies and claims at once.
Needs Streamlit 1.35+ (row selection, segmented control with fallback).

## Customer 360
Opening the page shows a customer directory (cards with properties, policies, premium and flags such as "Renews in 40d" or "Open claims"), with quick views, sorting and the header search.
Click a card to open that customer's full 360 view; "← All customers" returns to the directory.
