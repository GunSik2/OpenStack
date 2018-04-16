
# Backup
- Backup
```
pg_dump -U username -W -F t database_name > backup_file.tar
```
- Restore
```
pg_restore --dbname=database_name --create --verbose backup_file.tar
```
- Restore
```
CREATE DATABASE database_name;
pg_restore --dbname=database_name --verbose c:\pgbackup\dvdrental.tar
```

## Reference
- http://www.postgresqltutorial.com/postgresql-backup-database/

