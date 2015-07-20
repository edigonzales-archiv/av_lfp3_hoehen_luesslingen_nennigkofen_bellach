# LFP3-Höhen aus alten ITF

## Schema in DB erzeugen

```
#!psql
CREATE SCHEMA av_lfp3_tmp
  AUTHORIZATION stefan;
GRANT ALL ON SCHEMA av_lfp3_tmp TO stefan;
GRANT USAGE ON SCHEMA av_lfp3_tmp TO mspublic;
```

## ITF importieren

Es wurde das jüngste ITF importiert, das noch LFP3-Höhen hatte:

* Bellach_011122.itf 
* Luesslingen34_001002.itf
* Nennigkofen34_001002.itf


```
#!bash
java -jar ili2pg.jar --import --dbhost localhost --dbport 5432 --dbdatabase rosebud2 --dbschema av_lfp3_tmp --dbusr stefan --dbpwd ziegler12 --models Grunddatensatz_aV93SO --modeldir http://www.catais.org/models/ --sqlEnableNull --createGeomIdx --nameByTopic --strokeArcs --createEnumTxtCol --skipPolygonBuilding ../itf/Bellach_011122.itf
```

Der Import von `Bellach_011122.itf` brach ab. Aus diesem Grund wurde die Option `--skipPolygonBuilding` verwendet. Nennigkofen Los 5 wurde nicht importiert (kleines GZ-Gebiet ausserhalb Baugebiet).

## Zuweisung LFP3 aktuell - LFP3 alt

Wie viele der alten LFP3, die eine Höhe aufweisen, können punktgenau (identische Koordinate) den aktuellen LFP3 zugewiesen werden?

```
#!psql
CREATE OR REPLACE VIEW av_lfp3_tmp.alt_vs_neu AS

SELECT t_id, alt.nummer as nr_alt, neu.nummer as nr_neu, alt.hoehegeom as hoehe_alt, neu.hoehegeom as hoehe_neu,
       alt.punktzeichen_txt as pz_alt, neu.punktzeichen_txt as pz_neu, 
       to_date(neu.gueltigereintrag, 'YYYYMMDD') as gueltigereintrag, to_date(neu.datum1, 'YYYYMMDD') as datum1, 
       neu.geometrie, (alt.hoehegeom - neu.hoehegeom)*100 as dh_cm
FROM
(
 SELECT t_id, nummer, hoehegeom, punktzeichen_txt, lagegeom
 FROM av_lfp3_tmp.fixpunkte_lfp3
) as alt,
(
 SELECT *
 FROM 
 (
  SELECT entstehung, nummer, hoehegeom, punktzeichen_txt, geometrie 
  FROM av_avdpool_ng.fixpunktekategorie3_lfp3
  WHERE gem_bfs IN (2464, 2542)
 ) a,
 (
  SELECT tid, gueltigereintrag, datum1
  FROM av_avdpool_ng.fixpunktekategorie3_lfp3nachfuehrung
  WHERE gem_bfs IN (2464, 2542)
 ) b
 WHERE a.entstehung = b.tid
) as neu
WHERE alt.lagegeom && neu.geometrie
AND ST_Equals(ST_AsBinary(alt.lagegeom), ST_AsBinary(neu.geometrie));
```

Bei insgesamt 1258 neuen und 1222 alten LFP3 gibts aufgrund der identischen Koordinate 1062 Übereinstimmungen. Aktuelle LFP3, die keine Höhen zugewiesen bekommen:

```
#!psql
CREATE OR REPLACE VIEW av_lfp3_tmp.neu_ohne_alte_hoehe AS

SELECT neu.*
FROM
(
 SELECT ogc_fid, nummer, hoehegeom, geometrie
 FROM av_avdpool_ng.fixpunktekategorie3_lfp3
 WHERE gem_bfs IN (2464, 2542)
) as neu,
(
 SELECT (ST_DumpPoints(ST_Difference(neu.geometrie, alt.lagegeom))).geom as geom
 FROM
 (
  SELECT ST_Union(lagegeom) as lagegeom
  FROM av_lfp3_tmp.fixpunkte_lfp3
  WHERE hoehegeom IS NOT NULL
 ) as alt,
 (
  SELECT ST_Union(geometrie ) as geometrie
  FROM av_avdpool_ng.fixpunktekategorie3_lfp3
  WHERE gem_bfs IN (2464, 2542)
 ) as neu
 WHERE alt.lagegeom && neu.geometrie
) as diff
WHERE neu.geometrie && diff.geom
AND ST_Equals(ST_AsBinary(neu.geometrie), ST_AsBinary(diff.geom));
```

## Vergleich alte LFP3-Höhen mit DTM (LiDAR 2014)

```
#!psql
CREATE OR REPLACE VIEW av_lfp3_tmp.alt_vs_dtm AS
 
SELECT p.t_id, p.nr_alt, p.nr_neu, p.h_alt, p.h_neu, p.pz_alt, p.pz_neu, 
       p.gueltigereintrag, p.datum1, p.geometrie, ST_Value(r.rast, p.geometrie) as h_dtm,
       (ST_Value(r.rast, p.geometrie) - p.h_alt)*100 as dh_cm
FROM av_lfp3_tmp.neu_mit_alter_hoehe as p, av_lidar_2014.dtm as r
WHERE ST_Intersects(r.rast, p.geometrie);
```

## Plausibilitätskontrolle

### Vergleich DTM mit LFP3-Höhen in Zuchwil und Biberist

```
#!psql
SELECT p.ogc_fid, p.nummer, p.kategorie, p.geometrie, ST_X(p.geometrie) as x, ST_Y(p.geometrie) as y,
   p.hoehe as h_lfp, ST_Value(r.rast, p.geometrie) as h_dtm,
   (ST_Value(r.rast, p.geometrie) - p.hoehe)*100 as dh_cm, p.bfsnr
FROM 
(
SELECT * 
FROM av_avwmsde_t.cppt
WHERE bfsnr IN (2534, 2513)
) as p, av_lidar_2014.dtm as r
WHERE ST_Intersects(r.rast, p.geometrie)
AND p.hoehe IS NOT NULL
AND p.kategorie IN ('LFP1', 'LFP2', 'LFP3');
```


