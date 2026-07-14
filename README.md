# TechCare Solutions Riwi S.A.S. — Relational Database Project

## Project description

TechCare Solutions Riwi S.A.S. provides preventive and corrective maintenance
services for technology equipment to client companies across several cities.
All operational data (clients, technicians, equipment, services, branches and
cities) used to be tracked in a single shared Excel file, which caused
duplicated clients, inconsistent branch and city names, repeated technicians,
and unreliable reports.

This project analyzes the original Excel file, removes redundancies and
inconsistencies, normalizes the data up to Third Normal Form (3FN), and
implements the resulting model as a relational database, together with the
scripts needed to create it, load it, manipulate it, and query it for real
business needs.

## Technologies used

- MySQL 8.0 (SQL DDL / DML / DQL)
- Microsoft Excel (.xlsx) for the normalization workflow
- Mermaid.js (`erDiagram`) for the Entity-Relationship Diagram

## Database engine

MySQL.

## Explanation of the normalization process

**Initial state.** The source file (`Dataset_TechCareSolutions_jonada_intermedia.xlsx`)
contains 20 flat rows mixing client, city, branch, technician, equipment,
equipment category, service type, work order, date, hours and cost in a
single table.

**Problems found:**
- The same client, city, branch, technician, equipment, category and service
  type appear written in multiple ways (e.g. `Acme Ltd` / `ACME LTDA`,
  `Bogotá` / `Bogota`, `J. Perez` / `Juan Perez`, `Preventive` /
  `Preventive Maintenance`).
- City, category and service-type information was repeated on every row
  instead of being stored once.
- No primary/foreign keys, so referential integrity could not be enforced.

**1FN (First Normal Form).** All values were standardized to a single
canonical spelling per entity, so every cell holds one atomic value (see
sheet `02_1FN` in the normalized Excel file). At this stage the data is still
a single flat table.

**2FN (Second Normal Form).** Attributes that depend only on part of a
composite identifier (e.g. a branch's city, an equipment's category) were
extracted into their own tables, removing partial dependencies.

**3FN (Third Normal Form).** Transitive dependencies were removed by
splitting the flat table into independent catalog tables — `riwi_cities`,
`riwi_branches`, `riwi_clients`, `riwi_technicians`,
`riwi_equipment_categories`, `riwi_equipment`, `riwi_service_types` — plus one
fact table, `riwi_service_orders`, whose non-key attributes depend only on its
own primary key and reference every catalog through a foreign key. The 7
different service-type labels in the source file were consolidated into 3
real business service types: `Preventive Maintenance`, `Corrective
Maintenance`, `Installation`.

The full step-by-step transformation (dirty data → 1FN → each 3FN table) is
documented in `Dataset_TechCareSolutions_normalizado.xlsx`.

## Database structure

| Table | Description | Key columns |
|---|---|---|
| `riwi_cities` | City catalog | `id` PK |
| `riwi_branches` | Service branch catalog | `id` PK, `city_id` FK |
| `riwi_clients` | Client catalog | `id` PK |
| `riwi_technicians` | Technician catalog | `id` PK |
| `riwi_equipment_categories` | Equipment category catalog | `id` PK |
| `riwi_equipment` | Equipment catalog | `id` PK, `category_id` FK |
| `riwi_service_types` | Service type catalog | `id` PK |
| `riwi_service_orders` | Fact table: one row per service order | `id` PK, `client_id`, `branch_id`, `technician_id`, `equipment_id`, `service_type_id` FK |

All foreign keys are declared `ON UPDATE CASCADE ON DELETE RESTRICT`, so a
catalog row cannot be deleted while any service order still references it.
`NOT NULL` and `UNIQUE` constraints protect required fields and prevent
duplicate catalog entries. `CHECK` constraints guarantee `hours > 0` and
`cost >= 0`.

## Entity Relationship Diagram

See `MER_TechCareSolutions.png`. It shows all 8 entities, their attributes,
primary/foreign keys and crow's-foot cardinalities (one catalog row relates
to many service orders).

## Database creation instructions

1. Open a MySQL client (MySQL Workbench, CLI, etc.) connected to a MySQL 8.0
   server.
2. Open `bd_ramon_vega_cumbia.sql` **completely** and execute it as a whole
   script (not statement by statement), since later tables reference earlier
   ones through foreign keys.
3. The script will:
   - Drop the database if it already exists, then create
     `bd_ramon_vega_cumbia`.
   - Create the 8 tables described above (DDL section).

## Data loading instructions

The same script (`bd_ramon_vega_cumbia.sql`) loads the data right after the
DDL section: 10 cities, 10 branches, 10 clients, 5 technicians, 4 equipment
categories, 6 pieces of equipment, 3 service types and 20 service orders,
using plain `INSERT` statements with explicit `id` values so every foreign
key relationship is predictable and verifiable. This approach was chosen
over CSV/`LOAD DATA` because the full data set is small (20 transactional
rows) and a single SQL script is easier to review, version and reproduce
end-to-end without depending on external file paths.

## Data manipulation scripts (DML)

Included in the same script, right after the data load:

- **Insert**: registers a new client (`TechnoWorld`) together with a new
  service order, inside a transaction (`START TRANSACTION` / `COMMIT`),
  using `LAST_INSERT_ID()` to link the new order to the new client.
- **Update**: updates an existing technician's full name.
- **Delete**: deletes an equipment record (`HP LaserJet`, loaded on purpose
  with no service orders) only if it has no associated service orders
  (`NOT EXISTS` check). The `ON DELETE RESTRICT` foreign key additionally
  guarantees that any equipment that does have service orders cannot be
  deleted, even by mistake — this was verified by attempting to delete an
  equipment in use, which MySQL correctly rejects with error 1451.

## Explanation of each SQL query

1. **Orders per technician** — counts how many service orders each
   technician has attended, so support coordination can balance workload.
2. **Service history per city** — joins city → branch → service order and
   counts orders per city, so regional managers can see where most services
   happen.
3. **Total services per service type** — counts orders per service type
   (preventive, corrective, installation), so operations can see which
   services are most requested.
4. **Equipment with the most maintenances** — counts orders per equipment,
   so support analysts can spot equipment needing frequent attention.
5. **Clients with the most service orders** — counts orders per client, so
   the commercial team can identify high-demand clients for loyalty
   strategies.
6. **Orders managed per branch** — counts orders per branch (with its city),
   so operations managers can plan staffing and resources.

All queries use `LEFT JOIN` where a catalog entry with zero orders should
still appear in the result (e.g. a technician or service type with no
orders yet).

## Developer information

- **Full name:** Ramon Vega
- **Clan:** Cumbia

## Files in this delivery

- `bd_ramon_vega_cumbia.sql` — DDL, data load, DML and SQL queries.
- `MER_TechCareSolutions.png` — Entity-Relationship Diagram.
- `Dataset_TechCareSolutions_normalizado.xlsx` — normalization evidence
  (dirty data → 1FN → 3FN tables).
- `Dataset_TechCareSolutions_jonada_intermedia.xlsx` — original source file.
- `README.md` — this document.
