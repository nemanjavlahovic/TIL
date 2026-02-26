# PostgreSQL indexes are B-tree by default

Every `@@index` and `@unique` in my Prisma schema creates a sorted lookup structure (B-tree) on that column. Without them, Postgres scans every row. Adding them to columns I filter/sort on is what makes queries fast. The B-tree part is just how Postgres organizes it internally — I don't choose it, it's the default.
