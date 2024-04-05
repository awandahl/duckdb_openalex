# duckdb_openalex

Playing around with DuckDB and OpenAlex. Much easier than I thought!    
**Big thanks** to https://github.com/chrisgebert/open_alex_snapshot

I'm running this with Ubuntu 22.04.1 on a Dell Optiplex 9020 (2013), Intel(R) Core(TM) i7-4770 CPU @ 3.40GHz, 24 GB RAM

### Getting the snapshot:

Read: https://docs.openalex.org/download-all-data/download-to-your-machine    

Install AWS client
````
sudo apt install awscli
````
About 300Gb which takes approximately three hours to download.

````
aws s3 sync "s3://openalex" "openalex-snapshot" --no-sign-request
````


### Open a new db

First get the DuckDB binary:  
````
wget https://github.com/duckdb/duckdb/releases/download/v0.10.1/duckdb_cli-linux-amd64.zip
````
then

````
./duckdb open_alex_test.duckdb
````

### Checking the size of authors = 90M+

````
select count(*)
from read_ndjson(
  '/home/aw/oal/openalex-snapshot/data/authors/*/*.gz'
);
````
````
┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│     90556187 │
└──────────────┘
````

### Create a table for the authors

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

### A test search, find 100 authors affiliated to KTH and create an additional column 'orcid_modified' for ORCiD where the number isn't prepended by 'https://orcid.org/'

````
SELECT *, REPLACE(orcid, 'https://orcid.org/', '') AS orcid_modified
FROM authors
WHERE last_known_institution.id = 'https://openalex.org/I86987016'
LIMIT 100;
````

### Write a parquet file for the same search

````

COPY (
    SELECT *, REPLACE(orcid, 'https://orcid.org/', '') AS orcid_modified
    FROM authors
    WHERE last_known_institution.id = 'https://openalex.org/I86987016'
    LIMIT 100
) TO '/home/aw/oal/kth_authors_oal_2024-04-05.parquet' (FORMAT 'parquet');

````

### Checking the size of institutions = 100K+

````
select count(*)
from read_ndjson(
  '/home/aw/oal/openalex-snapshot/data/institutions/*/*.gz'
);
````
````
┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│       107716 │
└──────────────┘
````

### Create a table for institutions (not yet exhaustive)

````

CREATE TABLE institutions AS
SELECT *
FROM read_ndjson(
'/home/aw/oal/openalex-snapshot/data/institutions/*/*.gz', columns = {
    id: 'VARCHAR',
    display_name: 'VARCHAR',
    alternate_titles: 'VARCHAR',
    country_code: 'VARCHAR',
    type: 'VARCHAR',
    works_count: 'BIGINT',
    cited_by_count: 'BIGINT',
    homepage_url: 'VARCHAR',
    ids: 'STRUCT(openalex VARCHAR, ror VARCHAR, mag VARCHAR, wikidata VARCHAR, grid VARCHAR)',
    updated_date: 'DATE',
    created_date: 'DATE',
    updated: 'VARCHAR'
  },
  compression='gzip'
);
````

### A test search, show 100 Swedish institutions and all available columns, ordered by number of works

````
SELECT *
FROM institutions
WHERE country_code = 'SE'
ORDER BY works_count DESC
LIMIT 100;
````

### Checking the size of funders = 32K+

````
select count(*)
from read_ndjson(
  '/home/aw/oal/openalex-snapshot/data/funders/*/*.gz'
);
````
````
┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│        32437 │
└──────────────┘

````

### Create a table for funders (not yet exhaustive)

````
CREATE TABLE funders AS
SELECT *
FROM read_ndjson(
'/home/aw/oal/openalex-snapshot/data/funders/*/*.gz', columns = {
    id: 'VARCHAR',
    display_name: 'VARCHAR',
    alternate_titles: 'VARCHAR',
    country_code: 'VARCHAR',
    grants_count: 'BIGINT',
    works_count: 'BIGINT',
    cited_by_count: 'BIGINT',
    ids: 'STRUCT(openalex VARCHAR, ror VARCHAR, wikidata VARCHAR, crossref VARCHAR, doi VARCHAR)',
    updated_date: 'DATE',
    created_date: 'DATE',
    updated: 'VARCHAR'
  },
  compression='gzip'
);
````

### A test search, showing 100 Swedish funders and all available columns, ordered by number of grants 

````
SELECT *
FROM funders
WHERE country_code = 'SE'
ORDER BY grants_count DESC
LIMIT 100;
````


https://www.christophenicault.com/post/large_dataframe_arrow_duckdb/
