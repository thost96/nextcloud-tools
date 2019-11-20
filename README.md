# nextcloud-tools

The **nextcloud-tools** have been developed by [SysEleven](https://syseleven.de) to debug the encryption and signature methods executed by the default encryption module of Nextcloud. The provided scripts may be helpful for other Nextcloud users and developers to debug problems or restore files.

## check-signature.php

The `check-signature.php` script contains a re-implementation of Nextcloud's signature checking process. It supports different types of private keys - including master keys, public sharing keys, recovery keys and user keys. Furthermore, it supports different types of files - including regular files, version files, trashed files and trashed version files.

### Preparation

As the `check-signature.php` script does not implement database accesses, the necessary Nextcloud database tables have to be provided in the form of well-structured CSV files. These files can be exported directly from the database.

#### MariaDB/MySQL

To export the necessary CSV files from MariaDB/MySQL you have to connect to the correct database: `sudo mysql -D <dbname>`

Then you can execute the export:

```
SELECT storage, path, encrypted FROM oc_filecache INTO OUTFILE '/var/lib/mysql-files/filecache.csv' FIELDS ESCAPED BY '' TERMINATED BY ',' LINES TERMINATED BY '\n';

SELECT numeric_id, id FROM oc_storages INTO OUTFILE '/var/lib/mysql-files/storages.csv' FIELDS ESCAPED BY '' TERMINATED BY ',' LINES TERMINATED BY '\n';

QUIT;
```

You finally have to move the CSV files to their target location:

```
sudo mv /var/lib/mysql-files/filecache.csv /tmp/

sudo mv /var/lib/mysql-files/storages.csv /tmp/
```

#### PostgreSQL

To export the necessary CSV files from PostgreSQL you have to connect to the correct database: `sudo -u <dbuser> psql -d <dbname>`

Then you can execute the export:

```
\COPY (SELECT storage, path, encrypted FROM oc_filecache) TO '/tmp/filecache.csv' WITH CSV DELIMITER ',';

\COPY (SELECT numeric_id, id FROM oc_storages) TO '/tmp/storages.csv' WITH CSV DELIMITER ',';

\q
```

### Configuration

The `check-signature.php` script needs some configuration values to be set.

#### Nextcloud Definitions

The Nextcloud definitions are values that you have to copy from the Nextcloud configuration file (`"config/config.php"`). The names of the values are equal to the ones found in the Nextcloud configuration file.

#### Custom Definitions

The custom definitions define how the `check-signature.php` script works internally. These are the supported configuration values:

* `DEBUGLEVEL` - defines how much output is generated by the script. `DEBUG_DEFAULT` only outputs negative results. `DEBUG_INFO` outputs negative and positive results. `DEBUG_DEBUG` outputs negative and positive results as well as details about the internal state of the script. (*Use `DEBUG_DEBUG` with caution as it can produce a lot of output.*)
* `FILECACHE` - defines the path of the CSV export of the `oc_filecache` table.
* `FIXSIGNATURES` - defines whether files with bad signatures shall be fixed. `FIX_NONE` disables this feature. `FIX_DATABASE` tries to generate SQL statements to fix the database entries of the files. `FIX_FILE` tries to rewrite the files with correct signatures. (*Use `FIX_FILE` with caution as it can break your files.*)
* `KEYTYPE` - defines which key type shall be used to decrypt file keys. `KEY_MASTER` activates the master key support. `KEY_PUBSHARE` activates the public sharing key support. `KEY_RECOVERY` activates the recovery key support. `KEY_USER` activates the user key support.
* `MAXFILESIZE` - defines the maximum size of handled files in bytes. Set the memory limit accordingly: `php -d memory_limit=<2 * MAXFILESIZE + sizeof(/tmp/filecache.txt) + sizeof(/tmp/storages.txt) + overhead> ./check-signature.php`
* `MAXVERSION` - defines up to which version number signatures shall be checked when `FIXSIGNATURE` is set to `FIX_DATABASE`.
* `RECOVERY_PASSWORD` - defines the password of the recovery key when the recovery key support is activated through `KEYTYPE` or when the signature of the recovery private key shall be checked.
* `STORAGES` - defines the path of the CSV export of the `oc_storages` table.
* `USER_NAME` - defines the name of the user key when the user key support is activated through `KEYTYPE`. 
* `USER_PASSWORD` - defines the password of the user key when the user key support is activated through `KEYTYPE` or when the signature of the user private key of `USER_NAME` shall be checked.

#### User Password Definitions

The `check-signature.php` script supports to check the signature of user key files. In order to do this the scripts needs to have acces to the password of the user key. To provide the passwords for different users the corresponding `USER_PASSWORD_USERNAME` value can be set whereby `USERNAME` has to be replaced with the actual username.

### Execution

The `check-signature.php` script supports two different ways of execution.

#### Check individual files

The `check-signature.php` script supports to check individual files. In order to do this the script has to be called with the names of the files that shall be checked. The files have to be referenced with their absolute path or with their path relative to the Nextcloud `datadirectory` folder.

#### Check the whole data directory

The `check-signature.php` scripts supports to check the whole Nextcloud data directory. In order to do this the script has to be called without additional parameters.

## decrypt-file.php

The `decrypt-file.php` script contains a re-implementation of Nextcloud's file decryption process. It supports different types of private keys - including master keys, public sharing keys, recovery keys and user keys. Furthermore, it supports different types of files - including regular files, version files, trashed files and trashed version files.

### Configuration

The `decrypt-file.php` script needs some configuration values to be set.

#### Nextcloud Definitions

The Nextcloud definitions are values that you have to copy from the Nextcloud configuration file (`"config/config.php"`). The names of the values are equal to the ones found in the Nextcloud configuration file.

#### Custom Definitions

The custom definitions define how the `decrypt-file.php` script works internally. These are the supported configuration values:

* `DEBUGLEVEL` - defines how much output is generated by the script. `DEBUG_DEFAULT` and `DEBUG_INFO` only output the decrypted file content and negative results. `DEBUG_DEBUG` outputs the decrypted file content and negative results as well as details about the internal state of the script. (*Use `DEBUG_DEBUG` with caution as it can produce a lot of output.*)
* `KEYTYPE` - defines which key type shall be used to decrypt file keys. `KEY_MASTER` activates the master key support. `KEY_PUBSHARE` activates the public sharing key support. `KEY_RECOVERY` activates the recovery key support. `KEY_USER` activates the user key support.
* `RECOVERY_PASSWORD` - defines the password of the recovery key when the recovery key support is activated through `KEYTYPE`.
* `USER_NAME` - defines the name of the user key when the user key support is activated through `KEYTYPE`. 
* `USER_PASSWORD` - defines the password of the user key when the user key support is activated through `KEYTYPE`.

### Execution

The `decrypt-file.php` script only supports to decrypt one individual file at a time. In order to do this the script has to be called with the name of the file that shall be decrypted. The file has to be referenced with its absolute path or with its path relative to the Nextcloud `datadirectory` folder.

The script outputs the decrypted file content to STDOUT so it is advised to pipe the output into a file.

## fix-duplicate.php

The `fix-signature.php` script checks the `oc_filecache` table for file duplicates and creates SQL statements to delete the duplicates. The problem this scripts tries to solve is that Nextcloud at some point in time added files to the `oc_filecache` table with two different entry structures:

1. `storage = 1` and `path = $username/$path`
2. `storage != 1` and `path = $path`

### Preparation

As the `fix-duplicate.php` script does not implement database accesses, the necessary Nextcloud database tables have to be provided in the form of well-structured CSV files. These files can be exported directly from the database.

#### MariaDB/MySQL

To export the necessary CSV files from MariaDB/MySQL you have to connect to the correct database: `sudo mysql -D <dbname>`

Then you can execute the export:

```
SELECT storage, path, encrypted FROM oc_filecache INTO OUTFILE '/var/lib/mysql-files/filecache.csv' FIELDS ESCAPED BY '' TERMINATED BY ',' LINES TERMINATED BY '\n';

SELECT numeric_id, id FROM oc_storages INTO OUTFILE '/var/lib/mysql-files/storages.csv' FIELDS ESCAPED BY '' TERMINATED BY ',' LINES TERMINATED BY '\n';

QUIT;
```

You finally have to move the CSV files to their target location:

```
sudo mv /var/lib/mysql-files/filecache.csv /tmp/

sudo mv /var/lib/mysql-files/storages.csv /tmp/
```

#### PostgreSQL

To export the necessary CSV files from PostgreSQL you have to connect to the correct database: `sudo -u <dbuser> psql -d <dbname>`

Then you can execute the export:

```
\COPY (SELECT storage, path, encrypted FROM oc_filecache) TO '/tmp/filecache.csv' WITH CSV DELIMITER ',';

\COPY (SELECT numeric_id, id FROM oc_storages) TO '/tmp/storages.csv' WITH CSV DELIMITER ',';

\q
```

### Configuration

The `fix-duplicate.php` script needs some configuration values to be set.

#### Nextcloud Definitions

The Nextcloud definitions are values that you have to copy from the Nextcloud configuration file (`"config/config.php"`). The names of the values are equal to the ones found in the Nextcloud configuration file.

#### Custom Definitions

The custom definitions define how the `fix-duplicate.php` script works internally. These are the supported configuration values:

* `DEBUGLEVEL` - defines how much output is generated by the script. `DEBUG_DEFAULT` and `DEBUG_INFO` only output positive and negative results. `DEBUG_DEBUG` outputs positive and negative results as well as details about the internal state of the script. (*Use `DEBUG_DEBUG` with caution as it can produce a lot of output.*)
* `FILECACHE` - defines the path of the CSV export of the `oc_filecache` table.
* `STORAGES` - defines the path of the CSV export of the `oc_storages` table.

### Execution

The `fix-duplicate.php` script does not support any parameters.

The script outputs the result to STDOUT so it is advised to pipe the output into a file.

## inject-content.php

The `inject-content.php` script takes a filename, a known plaintext and a target plaintext and manipulates a server-side encrypted file so that it contains the target plaintext instead of the known plaintext.

### Configuration

The `inject-content.php` script needs some configuration values to be set.

#### Nextcloud Definitions

The Nextcloud definitions are values that you have to copy from the Nextcloud configuration file (`"config/config.php"`). The names of the values are equal to the ones found in the Nextcloud configuration file.

#### Custom Definitions

The custom definitions define how the `inject-content.php` script works internally. These are the supported configuration values:

* `DEBUGLEVEL` - defines how much output is generated by the script. `DEBUG_DEFAULT` and `DEBUG_INFO` only output positive and negative results. `DEBUG_DEBUG` outputs positive and negative results as well as details about the internal state of the script. (*Use `DEBUG_DEBUG` with caution as it can produce a lot of output.*)
* `MAXFILESIZE` - defines the maximum size of handled files in bytes. Set the memory limit accordingly: `php -d memory_limit=<MAXFILESIZE + overhead> ./inject-content.php`

### Execution

The `inject-content.php` script only supports to inject one individual file at a time. In order to do this the script has to be called with the name of the file that shall be injected. The file has to be referenced with its absolute path or with its path relative to the Nextcloud `datadirectory` folder.

Additionally, the old content has be provided as a hexadecimally encoded value as well as the new content which also has to be provided as a hexadecimally encoded value. Both values must have the same length and their payload must not exceed 16 bytes.

The script outputs the result to STDOUT so it is advised to pipe the output into a file.

## strip-signature.php

The `strip-signature.php` script takes a filename and manipulates a server-side encrypted file so that its content blocks do not contain message authentication codes anymore.

### Configuration

The `strip-signature.php` script needs some configuration values to be set.

#### Nextcloud Definitions

The Nextcloud definitions are values that you have to copy from the Nextcloud configuration file (`"config/config.php"`). The names of the values are equal to the ones found in the Nextcloud configuration file.

#### Custom Definitions

The custom definitions define how the `strip-signature.php` script works internally. These are the supported configuration values:

* `DEBUGLEVEL` - defines how much output is generated by the script. `DEBUG_DEFAULT` and `DEBUG_INFO` only output positive and negative results. `DEBUG_DEBUG` outputs positive and negative results as well as details about the internal state of the script. (*Use `DEBUG_DEBUG` with caution as it can produce a lot of output.*)
* `MAXFILESIZE` - defines the maximum size of handled files in bytes. Set the memory limit accordingly: `php -d memory_limit=<MAXFILESIZE + overhead> ./strip-signature.php`

### Execution

The `strip-signature.php` script only supports to inject one individual file at a time. In order to do this the script has to be called with the name of the file that shall be injected. The file has to be referenced with its absolute path or with its path relative to the Nextcloud `datadirectory` folder.

Additionally, the old known content has to be provided as a hexadecimally encoded value as well as the new target content which also has to be provided as a hexadecimally encoded value. Both values must have the same length and their payload must not exceed 16 bytes.

The script outputs the result to STDOUT so it is advised to pipe the output into a file.

## server-side-encryption.md

The document `server-side-encryption.md` contains the collected knowledge of [SysEleven](https://syseleven.de) about the file types and key types as well as the key pair generation, file encryption and file decryption processes of Nextcloud.
