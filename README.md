# duckdb_openalex

Playing around with DuckDB and OpenAlex. Much easier than I thought!    
**Big thanks** to https://github.com/chrisgebert/open_alex_snapshot

I'm running this with Ubuntu 22.04.1 on a Dell Optiplex 9020 (2013), Intel(R) Core(TM) i7-4770 CPU @ 3.40GHz, 24 GB RAM

### Getting the snapshot:

Read: https://docs.openalex.org/download-all-data/download-to-your-machine  

AWS S3 Explorer:  https://openalex.s3.amazonaws.com/browse.html   

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
### Or rather only this, since the schema is identified rather well automagically:

````
CREATE TABLE authors AS
SELECT *
FROM read_ndjson(
'/home/aw/oal/openalex-snapshot/data/authors/*/*.gz', compression='gzip');

````
### Describe authors;   
will give this schema where last_known_instituion is JSON
````

┌──────────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬─────────┬─────────┬─────────┬─────────┐
│     column_name      │                                                                                column_type                                                                                 │  null   │   key   │ default │  extra  │
│       varchar        │                                                                                  varchar                                                                                   │ varchar │ varchar │ varchar │ varchar │
├──────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼─────────┼─────────┼─────────┼─────────┤
│ id                   │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ orcid                │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ display_name         │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ display_name_alter…  │ VARCHAR[]                                                                                                                                                                  │ YES     │         │         │         │
│ works_count          │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│ cited_by_count       │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│ most_cited_work      │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ summary_stats        │ STRUCT("2yr_mean_citedness" DOUBLE, h_index BIGINT, i10_index BIGINT, oa_percent DOUBLE, works_count BIGINT, cited_by_count BIGINT, "2yr_works_count" BIGINT, "2yr_cited…  │ YES     │         │         │         │
│ ids                  │ STRUCT(openalex VARCHAR, orcid VARCHAR, scopus VARCHAR)                                                                                                                    │ YES     │         │         │         │
│ last_known_institu…  │ JSON                                                                                                                                                                       │ YES     │         │         │         │
│ counts_by_year       │ STRUCT("year" BIGINT, works_count BIGINT, oa_works_count BIGINT, cited_by_count BIGINT)[]                                                                                  │ YES     │         │         │         │
│ x_concepts           │ STRUCT(id VARCHAR, wikidata VARCHAR, display_name VARCHAR, "level" BIGINT, score DOUBLE)[]                                                                                 │ YES     │         │         │         │
│ works_api_url        │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ updated_date         │ DATE                                                                                                                                                                       │ YES     │         │         │         │
│ created_date         │ DATE                                                                                                                                                                       │ YES     │         │         │         │
│ updated              │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
├──────────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴─────────┴─────────┴─────────┴─────────┤
│ 16 rows                                                                                                                                                                                                                         6 columns │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

````
### A test search, find 100 authors affiliated to KTH and create an additional column 'orcid_modified' for ORCiD where the number isn't prepended by 'https://orcid.org/'

````
SELECT *, REPLACE(orcid, 'https://orcid.org/', '') AS orcid_modified
FROM authors
WHERE last_known_institution::JSON->>'id' = 'https://openalex.org/I86987016'
ORDER BY works_count DESC
LIMIT 100;

SELECT id, last_known_institution::JSON->>'id' AS institution_id
FROM authors
WHERE last_known_institution::JSON->>'country_code' = 'SE'
LIMIT 100;



````

### Write a parquet file for the first test search above

````

COPY (
    SELECT *, REPLACE(orcid, 'https://orcid.org/', '') AS orcid_modified
    FROM authors
    WHERE last_known_institution::JSON->>'id' = 'https://openalex.org/I86987016'
    LIMIT 100
) TO '/home/aw/oal/kth_authors_oal_2024-04-10.parquet' (FORMAT 'parquet');

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

### Works.... testing a bit

````
CREATE TABLE works AS
    SELECT *
    FROM read_ndjson(
      '/home/aw/oal/openalex-snapshot/data/works/updated_date=2024-03-27/part_028.gz');
````
````
DESCRIBE works;
````
````

┌──────────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬─────────┬─────────┬─────────┬─────────┐
│     column_name      │                                                                                column_type                                                                                 │  null   │   key   │ default │  extra  │
│       varchar        │                                                                                  varchar                                                                                   │ varchar │ varchar │ varchar │ varchar │
├──────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼─────────┼─────────┼─────────┼─────────┤
│ id                   │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ doi                  │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ doi_registration_a…  │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ display_name         │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ title                │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ publication_year     │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│ publication_date     │ DATE                                                                                                                                                                       │ YES     │         │         │         │
│ language             │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ language_id          │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ ids                  │ STRUCT(openalex VARCHAR, doi VARCHAR, pmid VARCHAR, mag BIGINT, arxiv_id VARCHAR, pmcid VARCHAR)                                                                           │ YES     │         │         │         │
│ primary_location     │ STRUCT(source STRUCT(id VARCHAR, issn_l VARCHAR, issn VARCHAR[], display_name VARCHAR, publisher VARCHAR, host_organization VARCHAR, host_organization_name VARCHAR, hos…  │ YES     │         │         │         │
│ best_oa_location     │ STRUCT(source STRUCT(id VARCHAR, issn_l VARCHAR, issn VARCHAR[], display_name VARCHAR, publisher VARCHAR, host_organization VARCHAR, host_organization_name VARCHAR, hos…  │ YES     │         │         │         │
│ type                 │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ type_crossref        │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ type_id              │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ indexed_in           │ VARCHAR[]                                                                                                                                                                  │ YES     │         │         │         │
│ open_access          │ STRUCT(is_oa BOOLEAN, oa_status VARCHAR, oa_url VARCHAR, any_repository_has_fulltext BOOLEAN)                                                                              │ YES     │         │         │         │
│ authorships          │ STRUCT(author_position VARCHAR, author STRUCT(id VARCHAR, display_name VARCHAR, orcid VARCHAR), institutions STRUCT(id VARCHAR, display_name VARCHAR, ror VARCHAR, count…  │ YES     │         │         │         │
│ countries_distinct…  │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│ institutions_disti…  │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│          ·           │   ·                                                                                                                                                                        │  ·      │    ·    │    ·    │    ·    │
│          ·           │   ·                                                                                                                                                                        │  ·      │    ·    │    ·    │    ·    │
│          ·           │   ·                                                                                                                                                                        │  ·      │    ·    │    ·    │    ·    │
│ referenced_works_c…  │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│ sustainable_develo…  │ STRUCT(id VARCHAR, display_name VARCHAR, score DOUBLE)[]                                                                                                                   │ YES     │         │         │         │
│ keywords             │ STRUCT(keyword VARCHAR, score DOUBLE)[]                                                                                                                                    │ YES     │         │         │         │
│ grants               │ STRUCT(funder VARCHAR, funder_display_name VARCHAR, award_id VARCHAR)[]                                                                                                    │ YES     │         │         │         │
│ apc_list             │ STRUCT("value" BIGINT, currency VARCHAR, value_usd BIGINT, provenance VARCHAR)                                                                                             │ YES     │         │         │         │
│ apc_paid             │ STRUCT("value" BIGINT, currency VARCHAR, value_usd BIGINT, provenance VARCHAR)                                                                                             │ YES     │         │         │         │
│ cited_by_percentil…  │ STRUCT(min BIGINT, max BIGINT)                                                                                                                                             │ YES     │         │         │         │
│ related_works        │ VARCHAR[]                                                                                                                                                                  │ YES     │         │         │         │
│ abstract_inverted_…  │ JSON                                                                                                                                                                       │ YES     │         │         │         │
│ counts_by_year       │ JSON[]                                                                                                                                                                     │ YES     │         │         │         │
│ cited_by_api_url     │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ updated_date         │ DATE                                                                                                                                                                       │ YES     │         │         │         │
│ created_date         │ DATE                                                                                                                                                                       │ YES     │         │         │         │
│ updated              │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ authors_count        │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│ concepts_count       │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│ topics_count         │ BIGINT                                                                                                                                                                     │ YES     │         │         │         │
│ has_fulltext         │ BOOLEAN                                                                                                                                                                    │ YES     │         │         │         │
│ fulltext_origin      │ VARCHAR                                                                                                                                                                    │ YES     │         │         │         │
│ authorships_trunca…  │ BOOLEAN                                                                                                                                                                    │ YES     │         │         │         │
├──────────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴─────────┴─────────┴─────────┴─────────┤
│ 54 rows (40 shown)                                                                                                                                                                                                              6 columns │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
````

https://www.christophenicault.com/post/large_dataframe_arrow_duckdb/
