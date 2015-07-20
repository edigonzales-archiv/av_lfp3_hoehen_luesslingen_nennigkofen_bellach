# LPF3-Höhen aus alten ITF

## Schema in DB erzeugen

```
#!psql
CREATE SCHEMA av_lfp3_tmp
  AUTHORIZATION stefan;
GRANT ALL ON SCHEMA av_lfp3_tmp TO stefan;
GRANT USAGE ON SCHEMA av_lfp3_tmp TO mspublic;
```


