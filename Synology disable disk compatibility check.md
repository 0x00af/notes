# How to disable disk compatibility check


## disable compatibility check

- Might stop working in a future DSM release.
- The DSM update might overwrite this file, just re-apply the change

```
vi /etc.defaults/synoinfo.conf
```

Set `support_disk_compatibility="no"`


## Manually edit the compatibility database

The database (a JSON file) file is located here:
```
/var/lib/disk-compatibility/<model>_host_v7.db
```

## Use 007revad/Synology_HDD_db

Use [007revad/Synology_HDD_db](https://github.com/007revad/Synology_HDD_db) to script the changes.


