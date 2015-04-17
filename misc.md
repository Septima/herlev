
```sql
CREATE DATABASE herlev
  WITH OWNER = postgres
       ENCODING = 'UTF8'
       TABLESPACE = pg_default
       LC_COLLATE = 'da_DK.utf8'
       LC_CTYPE = 'da_DK.utf8'
       CONNECTION LIMIT = -1;
GRANT CONNECT, TEMPORARY ON DATABASE data TO public;
GRANT ALL ON DATABASE data TO postgres;
GRANT CONNECT, CREATE ON DATABASE data TO septima;
GRANT TEMPORARY ON DATABASE data TO septima WITH GRANT OPTION;


DROP SCHEMA IF EXISTS tmp;
CREATE SCHEMA tmp
  AUTHORIZATION septima;

GRANT ALL ON SCHEMA tmp TO septima;
GRANT USAGE ON SCHEMA tmp TO septima;


ALTER DEFAULT PRIVILEGES IN SCHEMA tmp
    GRANT SELECT, REFERENCES, TRIGGER ON TABLES
    TO septima;

ALTER DEFAULT PRIVILEGES IN SCHEMA tmp
    GRANT EXECUTE ON FUNCTIONS
    TO septima;

DROP SCHEMA IF EXISTS gartnerudbud;
CREATE SCHEMA gartnerudbud;

GRANT ALL ON SCHEMA gartnerudbud TO septima;
GRANT USAGE ON SCHEMA gartnerudbud TO septima;


ALTER DEFAULT PRIVILEGES IN SCHEMA gartnerudbud
    GRANT SELECT, REFERENCES, TRIGGER ON TABLES
    TO septima;

ALTER DEFAULT PRIVILEGES IN SCHEMA gartnerudbud
    GRANT EXECUTE ON FUNCTIONS
    TO septima;
/Library/Frameworks/GDAL.framework/Versions/1.11/Programs/ogr2ogr -f "postgresql" pg:"host=pg1.septima.dk user=martin dbname=herlev" Arbejdssted.TAB -lco SCHEMA=tmp -overwrite -lco OVERWRITE=YES

/Library/Frameworks/GDAL.framework/Versions/1.11/Programs/ogr2ogr -f "postgresql" pg:"host=pg1.septima.dk user=martin dbname=herlev" Linier.TAB -lco SCHEMA=tmp -overwrite -lco OVERWRITE=YES -nlt PROMOTE_TO_MULTI


/Library/Frameworks/GDAL.framework/Versions/1.11/Programs/ogr2ogr -f "postgresql" pg:"host=pg1.septima.dk user=martin dbname=herlev" Punkter.TAB -lco SCHEMA=tmp -overwrite -lco OVERWRITE=YES -nlt PROMOTE_TO_MULTI



DROP TABLE IF EXISTS gartnerudbud.merged;
CREATE TABLE gartnerudbud.merged AS
 
WITH alt AS (
SELECT 'Flader.tab' as origin, wkb_geometry, element, elementnavn, underelement, vedlniveau, 
       arealtype, CASE WHEN arbejdsstednr = '' THEN null
		        ELSE arbejdsstednr::integer
		        END	
  FROM tmp.flader 
  UNION
SELECT 'Punkter.tab' as origin, wkb_geometry, element, elementnavn, underelement, vedlniveau, 
       arealtype, arbejdsstednr::integer
  FROM tmp.punkter
  UNION
  SELECT 'Linjer.tab' as origin, wkb_geometry, element, elementnavn, underelement, vedlniveau, 
       arealtype, arbejdsstednr::integer
  FROM tmp.linier
  ),
  arbejdsted AS (
	SELECT arbejdsstednr, wkb_geometry as wkb_geometry FROM tmp.arbejdssted
	
  )
SELECT row_number() over () as id,  alt.* FROM alt;
ALTER TABLE gartnerudbud.merged ADD CONSTRAINT arbejdssted_pkey PRIMARY KEY (id);

ALTER TABLE gartnerudbud.merged
  OWNER TO herlev;
  
UPDATE gartnerudbud.merged merged
	SET arbejdsstednr = (SELECT arbejdsstednr::integer FROM tmp.arbejdssted as arb WHERE ST_Contains(arb.wkb_geometry, merged.wkb_geometry))
       WHERE arbejdsstednr is null;




```
