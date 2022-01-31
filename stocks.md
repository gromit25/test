```sql
CREATE TABLE "TestSchema".app_config (
	app_name varchar NOT NULL,
	config_key varchar NOT NULL,
	value varchar NULL,
	CONSTRAINT app_config_pk PRIMARY KEY (app_name, config_key)
);
```

```sql
CREATE TABLE "TestSchema".data_sources (
	alias varchar NOT NULL,
	org varchar NOT NULL,
	bucket varchar NOT NULL,
	tag varchar NOT NULL,
	field varchar NOT NULL,
	CONSTRAINT data_sources_pk PRIMARY KEY (alias)
);
```
