##Flader.tab

Indlæs fil i PostGIS

```bash
ogr2ogr -f "postgresql" pg:"host=127.0.0.1 password="hemmelig" user=mbj dbname=herlev" Flader.TAB -lco SCHEMA=tmp -overwrite -lco OVERWRITE=YES -a_srs epsg:25832 -nlt PROMOTE_TO_MULTI
```

Opdatér geomtrioplysninger

```sql
ALTER TABLE tmp.flader
  ALTER COLUMN wkb_geometry TYPE geometry(MULTIPOLYGON, 25832)
    USING ST_SetSRID(wkb_geometry,25832);
```
To geomtrier klarer ikke buffer0.cDerfor håndteres de særskilt.

```sql

--fix 1222: polygonen afsluttes ed at indsætte nyt punkt i start punkt og oprette polygonen

UPDATE tmp.flader
	SET wkb_geometry = ST_MakePolygon(ST_AddPoint(st_linemerge(wkb_geometry), ST_StartPoint(st_linemerge(wkb_geometry))))
	WHERE ogc_fid=1222;

--fix 1807:Buffer0 resulterer i null geometry. Vi anvender make valid og trækker kun polygoner ud
UPDATE tmp.flader
	SET wkb_geometry = ST_CollectionExtract(st_makevalid(wkb_geometry),3)
        WHERE ogc_fid =1807;
```


Opret ny tabel med buffer0 metoden på ulovlige geometrier
```sql


DROP TABLE IF EXISTS tmp.flader_buffer0;
CREATE TABLE tmp.flader_buffer0 AS
SELECT * FROM tmp.flader ORDER BY ogc_fid;

UPDATE tmp.flader_buffer0
 SET wkb_geometry = ST_Multi(st_buffer(wkb_geometry,0))
   WHERE not st_isvalid(wkb_geometry);

ALTER TABLE tmp.flader_buffer0
  ALTER COLUMN wkb_geometry TYPE geometry(MULTIPOLYGON, 25832)
    USING ST_SetSRID(wkb_geometry,25832);

ALTER TABLE tmp.flader_buffer0  ADD PRIMARY KEY (ogc_fid);
```

Eksporter til MI

```bash
ogr2ogr -f "MapInfo File" flader_buffer0.tab pg:"host=127.0.0.1 user=mbj password=hemmelig dbname=herlev" tmp.flader_buffer0
```

##Arbejdssted.tab

Indlæs fil i PostGIS

```bash
ogr2ogr -f "postgresql" pg:"host=127.0.0.1  password=hemmelig user=mbj dbname=herlev" Arbejdssted.TAB -lco SCHEMA=tmp -overwrite -lco OVERWRITE=YES
```
Opret ny tabel med buffer0 metoden
```sql
DROP TABLE IF EXISTS tmp.arbejdssted_buffer0;
CREATE TABLE tmp.arbejdssted_buffer0 AS
SELECT ogc_fid, ST_Multi(st_buffer(wkb_geometry,0)) as wkb_geometry, arbejdssted, arbejdsstednr, budgetomraade
  FROM tmp.arbejdssted;

ALTER TABLE IF EXISTS tmp.arbejdssteder_buffer0
  ALTER COLUMN wkb_geometry TYPE geometry(MULTIPOLYGON, 25832)
    USING ST_SetSRID(wkb_geometry,25832);

```
Eksporter arbejdssted til MI

```bash
 ogr2ogr -f "MapInfo File" arbejdssted_buffer0.tab pg:"host=127.0.0.1 user=mbj password=hemmelig dbname=herlev" tmp.arbejdssted_buffer0
```
