
with keys as (
	SELECT
		main.TABLE_CATALOG
		,main.TABLE_SCHEMA
		,main.TABLE_NAME
		,main.COLUMN_NAME
		,detail.CONSTRAINT_TYPE
	FROM
		AdventureWorksDW2017.INFORMATION_SCHEMA.CONSTRAINT_COLUMN_USAGE main
	JOIN
		AdventureWorksDW2017.INFORMATION_SCHEMA.TABLE_CONSTRAINTS detail
			on
				main.TABLE_CATALOG = detail.TABLE_CATALOG
				and
				main.TABLE_SCHEMA = detail.TABLE_SCHEMA
				and
				main.TABLE_NAME = detail.TABLE_NAME
				and
				main.CONSTRAINT_NAME = detail.CONSTRAINT_NAME
	where
		detail.CONSTRAINT_TYPE in ('PRIMARY KEY', 'UNIQUE')
),
survey as (
	  SELECT
	    detail.TABLE_SCHEMA resource_schema,
	    case
	      when main.TABLE_TYPE = 'VIEW' then 'view'
	      else 'table'
	    end resource_type,
	    detail.TABLE_NAME resource_name,
	    detail.ordinal_position resource_column_id,
	    detail.column_name resource_column_name,
	    case
	      when detail.is_nullable = 'NO' then 1
	      else 0
	    end notnull,
	    detail.data_type "type"
	  FROM
	    information_schema.tables main
	  JOIN (
	    select
	      col.TABLE_SCHEMA,
	      col.TABLE_NAME,
	      col.ordinal_position,
	      col.column_name,
	      col.data_type,
	      case
	        when col.character_maximum_length is not null then col.character_maximum_length
	        else col.numeric_precision
	      end as max_length,
	      col.is_nullable
	    from
	      information_schema.columns col
	    where
	      col.table_schema not in (
		      'sys',
		      'information_schema'
	      )
	    ) detail
	      on
	      main.TABLE_SCHEMA = detail.TABLE_SCHEMA
	      and
	      main.TABLE_NAME = detail.TABLE_NAME
	  WHERE
	    main.table_schema not in (
	      'sys',
	      'information_schema'
	    )
)
select
	keys.CONSTRAINT_TYPE
	,survey.*
from
	survey
left join keys
	on
		keys.TABLE_SCHEMA = survey.resource_schema  COLLATE DATABASE_DEFAULT
--		and
--		keys.TABLE_NAME = survey.resource_name  COLLATE DATABASE_DEFAULT
--		and
--		keys.COLUMN_NAME = survey.resource_column_name COLLATE DATABASE_DEFAULT
where keys.TABLE_SCHEMA is not null
		