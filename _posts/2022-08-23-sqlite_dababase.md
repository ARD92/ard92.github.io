---
layout: post
title: Setup and use Sqlite 
tags: linux
---

There are different types of databases that can be used. I initially used a JSON DB for simple file based databases while working on python based projects, but it was a little hard dealing with keys and querying them. SQLite solves all that.
its light weight, uses the same SQL query language like mysql, file based and is fast. Production grade as well for use cases like websites which don't need extremely high scale and resiliency. The sqlite official docs talks about the advantages and disadvantages of the same. 

## Setup SQLite3
```
apt install sqlite3
```
Intalling Sqlite will also give access to the python APIs too which could be leveraged in programs.

## Example 
[here](https://github.com/ARD92/BMPaaS/blob/main/manifests/consumer/sc-consumer.py) is an example how sqlite is used for storing discovered service chains over BGP. In this project, service chains are discovered over BMP and the parsed data is stored into different tables created on SQLite.

## Sqlalchemy 
pure pythonic way to handle queries. 
- [sql-alchemy](https://docs.sqlalchemy.org/en/20/)
- [sql-alchemy-git-link](https://github.com/sqlalchemy/sqlalchemy)

## References
- [when-to-use-sqlite](https://www.sqlite.org/whentouse.html)
- [backup-restore-sqlite](https://linuxhint.com/backup-restore-sqlite/)
- [recovering-corrupt-db](https://stackoverflow.com/questions/18259692/how-to-recover-a-corrupt-sqlite3-database)
