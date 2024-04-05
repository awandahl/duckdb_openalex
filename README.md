# duckdb_openalex
Playing around with DuckDB and OpenAlex


Checking the size of authors

````
select count(*)
from read_ndjson(
  '/home/aw/oal/openalex-snapshot/data/authors/*/*.gz'
);

┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│     90556187 │
└──────────────┘
````

Create a table for the authors

````

CREATE TABLE authors AS
SELECT *
FROM read_ndjson(
'/home/aw/oal/openalex-snapshot/data/authors/*/*.gz', columns = {
    id: 'VARCHAR',
    orcid: 'VARCHAR',
    display_name: 'VARCHAR',
    display_name_alternatives: 'VARCHAR[]',
    works_count: 'BIGINT',
    cited_by_count: 'BIGINT',
    most_cited_work: 'VARCHAR',
    ids: 'STRUCT(openalex VARCHAR, orcid VARCHAR, scopus VARCHAR)',
    last_known_institution: 'STRUCT(id VARCHAR, ror VARCHAR, display_name VARCHAR, country_code VARCHAR, type VARCHAR, lineage VARCHAR[])',
    counts_by_year: 'STRUCT(year VARCHAR, works_count BIGINT, oa_works_count BIGINT, cited_by_count BIGINT)[]',
    works_api_url: 'VARCHAR',
    updated_date: 'DATE',
    created_date: 'DATE',
    updated: 'VARCHAR'
  },
  compression='gzip'
);
````

SELECT *, REPLACE(orcid, 'https://orcid.org/', '') AS orcid_modified
FROM authors
WHERE last_known_institution.id = 'https://openalex.org/I86987016'
LIMIT 100;
````

COPY (
    SELECT *, REPLACE(orcid, 'https://orcid.org/', '') AS orcid_modified
    FROM authors
    WHERE last_known_institution.id = 'https://openalex.org/I86987016'
    LIMIT 100
) TO '/home/aw/oal/kth_authors_oal_2024-04-05.parquet' (FORMAT 'parquet');

````
