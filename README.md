# Korrektion af fejlbehæftede geometrier i park tabellerne



BLAH BLAH BLAHBLAH BLAHBLAHBLAHBLAHBLAHBLAHBLAH


 ## Overblikket

| Tabel          | Ingen fejl| Geometrifejl |SQL|
|-----------------|--------:|--------------:|:---:|
| Flader.tab      |    2073|      58      | <sup><sup>SELECT count(*), st_isvalid(wkb_geometry), ST_GeometryType(wkb_geometry) gtype FROM tmp.flader GROUP BY st_isvalid,gtype order by st_isvalid asc</sub></sub>|
| Linjer.Tab      |    180 |      0       | <sup><sup>SELECT count(*), st_isvalid(wkb_geometry), ST_GeometryType(wkb_geometry) gtype FROM tmp.linier GROUP BY st_isvalid,gtype order by st_isvalid asc</sub></sub>|
| Punkter.Tab     |   5428 |      0       | <sup><sup>SELECT count(*), st_isvalid(wkb_geometry), ST_GeometryType(wkb_geometry) gtype FROM tmp.punkter GROUP BY st_isvalid,gtype order by st_isvalid asc</sub></sub>|
| Arbejdssted.Tab |   109  |       15     | <sup><sup>SELECT count(*), st_isvalid(wkb_geometry), ST_GeometryType(wkb_geometry) gtype FROM tmp.arbejdssted GROUP BY st_isvalid,gtype order by st_isvalid asc</sub></sub>|

 I tabellen kan vi se, at det kun er nødvendigt at rette fejlene i tabellen **Flader.Tab** og **Arbejdssted.Tab**

## Flader

Tabellen kan indlæses i en PostgreSQL database med følgende kommando i kommandprompten

```bash
ogr2ogr -f "postgresql" pg:"host=pg1.septima.dk user=martin dbname=herlev" Flader.TAB -lco SCHEMA=tmp -overwrite -lco OVERWRITE=YES
```
Inden vi påbegynder oprettelsen fejlbehæftede geometrier skal vi have et overblik over antal af fejl fordelt på geometrityper:

```sql
SELECT count(*), st_isvalid(wkb_geometry), ST_GeometryType(wkb_geometry) gtype
 FROM tmp.flader
  GROUP BY st_isvalid,gtype
   order by st_isvalid asc
```
Hvilket giver følgende resultat:

|count|st_isvalid|gtype|
|------|---------|------|
|58|f|ST_MultiPolygon|
|2073|t|ST_MultiPolygon|


Altså har vi, at tabellen som udgangspunkt kun indeholder geometrier af typen **ST_MultiPolygon** og af dem er der 58, der returnerer *false* i testen [ST_Isvalid()](http://postgis.net/docs/ST_IsValid.html). Dermed skal vi forsøge at finde en metode til at gøre geometrierne gyldige ved at udføre forskellige manipulationer af geometrierne.

Vi kan også undersøge, hvilke typer af fejl geometrierne indeholder

```sql
SELECT ST_IsValidReason(wkb_geometry) as aarsag,ogc_fid as id
 FROM tmp.flader
 WHERE not st_isvalid(wkb_geometry)
```
Ovenstående SQL forespørgsel giver følgende resultat (trunkeret)

|aarsag|id|
|------|---|
|Self-intersection[715210.553283185 6181676.29453083]|3|
|Self-intersection[715437.904062876 6180562.61435407]|11|
|Self-intersection[713184.042703904 6183292.0187917]|19|
|Ring Self-intersection[713269.156308728 6183466.97318024]|18|
|--|--|

Vi kan lave en Mapinfo fil med resulaterne med ogr2ogr:

```bash
ogr2ogr -f "MapInfo File" flader_analyse.tab pg:"host=hostnavn/ip user=herlev password=secret dbname=herlev" -sql "SELECT st_isvalid(wkb_geometry), st_isvalidReason(wkb_geometry),*  from tmp.flader"
```


Vi kan se, at der er mange fejl af typerne **self-intersection** og **Ring Self-intersection**

PostGIS indeholder en [indbygget funktion](http://postgis.net/docs/ST_MakeValid.html), som forsøger at gøre geometrierne valide uden at fjerne *vertices*

```sql
SELECT count(*),
 st_isvalid(st_makevalid) st_isvalid,
 ST_GeometryType(st_makevalid) geomtype
  FROM (
  SELECT st_makevalid(wkb_geometry)
  FROM tmp.flader
  ) foo
GROUP BY st_isvalid, geomtype;
```

Vi kan se, at vi nu har gyldige geometrier, men at funktionen st_makevalid() har resulteret i at der nu er geometrityper, der ikke længere er af typen **ST_MultiPolygon**. Dem kan vi i det følgende forsøge at håndtere, men det gør opgaven yderliegere kompliceret.


|count|st_isvalid|geomtype|
|------|----------|---------|
|2099|t|ST_MultiPolygon|
|31|t|ST_GeometryCollection|
|1|t|ST_MultiLineString|

Vi kan også prøve med at lave en buffer på 0 meter. Det er en udbredt metode til at oprette polygoner med self intersections m.m.

```sql
SELECT count(*),
 st_isvalid(st_buffer) st_isvalid,
 ST_GeometryType(st_buffer) geomtype
  FROM (
  SELECT st_buffer(wkb_geometry,0)
  FROM tmp.flader
  ) foo
GROUP BY st_isvalid, geomtype;
```

Hvilket giver os et andet resultat uden geometrycollections

| count | st_isvalid |    geomtype     |
|-------|------------|-----------------|
|   220 | t          | ST_MultiPolygon|
|  1911 | t          | ST_Polygon|


I mappen "reports" er der lavet rapporter med den oprindelige geomtri og gemotrien efter en buffer 0 operation. Der skal tages stilling til for hvilke geometrier denne metode ikke skal bruges.
