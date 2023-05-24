#influxdb 

## Commands:

- access the CONSOLE interface on a Docker container:
```bash
docker exec -it influxdb /bin/bash
```

- access infludb console interface:
```bash
influx
```

- list databases:
```
show DATABASES
```

- create a new DATABASE:
```
create DATABASE [name]
```

-  access your database:
```
use [database name]
```

- list measurements of your database:
```
show measurements
```

- list field keys of your database:
```
show field keys
```

- create an entry on your database, defining the fields:
```
insert [measurement],field_name=value
```

- example to create a database and fields:
```
influx
show databases
create database THERMOSTAT
use THERMOSTAT
show measurements
insert stations,name=sensor1 temp=0.1,humidity=99.9
insert stations,name=sensor2 temp=0.1,humidity=99.9
insert stations,name=sensor3 temp=0.1,humidity=99.9
show measurements
select * from stations
```

- example to use SELECT and check the entries on your database:
```
select * from stations limit 10
select * from stations where "name"='sensor1' limit 10
select * from stations order by desc limit 10 (most recent)
select * from stations order by asc limit 10 (oldest first - default)
```
