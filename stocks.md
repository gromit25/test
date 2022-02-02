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
	alias_desc varchar NULL,
	org varchar NOT NULL,
	bucket varchar NOT NULL,
	tag varchar NOT NULL,
	CONSTRAINT data_sources_pk PRIMARY KEY (alias)
);
```

```sql
CREATE TABLE "TestSchema".data_fields (
	alias varchar NOT NULL,
	field_name varchar NOT NULL,
	field_desc varchar NULL,
	CONSTRAINT data_fields_pk PRIMARY KEY (alias,field_name),
	CONSTRAINT data_fields_fk FOREIGN KEY (alias) REFERENCES "TestSchema".data_sources(alias) ON DELETE CASCADE
);
```
