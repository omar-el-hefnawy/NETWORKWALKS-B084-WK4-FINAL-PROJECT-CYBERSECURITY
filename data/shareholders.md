# Exposed `shareholders` table — `mediroza_hr` database

Source: `mediroza_db_backup_2019.sql`, recovered via the `/old/` directory disclosed in `robots.txt` (see [Finding 2](../README.md#finding-2-exposed-database-backup-file-via-robotstxt)).

10 records, all fields as stored in the backup:

| ID | Shareholder | Share % | Shares held | Share class |
|---|---|---|---|---|
| 1 | Dr. Rajesh Naidoo | 18.0% | 180,000 | Ordinary |
| 2 | Cedar Health Holdings (Pty) Ltd | 15.0% | 150,000 | Ordinary |
| 3 | Dr. Johan van der Merwe | 12.0% | 120,000 | Ordinary |
| 4 | Reddy Family Trust | 11.0% | 110,000 | Ordinary |
| 5 | Thabo Molefe | 10.0% | 100,000 | Ordinary |
| 6 | Sarah Botha | 9.0% | 90,000 | Ordinary |
| 7 | Dr. Ahmed Kara | 8.0% | 80,000 | Preferential |
| 8 | Naledi Zulu | 7.0% | 70,000 | Ordinary |
| 9 | Michael Roberts | 6.0% | 60,000 | Ordinary |
| 10 | Dr. Vikram Chetty | 4.0% | 40,000 | Preferential |

**Why this matters:** cross-referencing this against the `staff` table shows that company ownership is concentrated among senior clinical/exec staff (the Chief Pathologist, Medical Director, CFO) plus two external entities (a holding company and a family trust) — and that a Network Engineer (ID 10, Thabo Molefe) and a Pharmacy Assistant (ID 16, Naledi Zulu) hold meaningful equity stakes despite comparatively junior listed salaries. This kind of corporate ownership/shareholding data is commercially sensitive and would normally never be stored in the same unauthenticated backup as HR records, let alone be web-reachable.
