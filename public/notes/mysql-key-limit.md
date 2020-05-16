After switching the default charset from `utf8` to `utf8mb4` to support emojis on [Directus](https://getdirectus.com), we started to receive errors that the key was too long. One my wondering how changing the charset affect the key length. Below can be see examples of the errors:

```
#1071 - Specified key was too long; max key length is 767 bytes
#1071 - Specified key was too long; max key length is 1000 bytes
#1071 - Specified key was too long; max key length is 3072 bytes
```

It can be any of previous errors depending on what is the storage engine of the table. MySIAM, InnoDb or InnoDb with `innodb_large_prefix` enabled have differents key length limitation.


## TL;DR

The difference between `utf8` and `utf8mb4` charset is the bytes requires to store each characters. `utf8` requires 3 bytes, while `utf8mb4` requires 4 bytes. This means using `utf8mb4` charset in a table with innodb engine with `innodb_large_prefix` disabled, at most 191 characters in a string column must be used.

191 characters × 4 bytes = 764 bytes which is less than the maximum length of 767 bytes allowed when `innodb_large_prefix` is disabled. Since MySQL 5.7 `innodb_large_prefix` is enabled by default allowing up to 3072 bytes.

## String Storage

String storage size vary depends on whether the column is fixed-length or variable-length. It also depends on the charset, it takes more bytes to storage a japanese character than an ASCII/Latin letter.

As an example, `CHAR` is a fixed-length while `VARCHAR` and `TEXT` are variable-length.

All fixed-length data types uses all the bytes they were declared. For example `CHAR(16)`, no matter what its value is, it's right padded with spaces to fill up to the specific length. On the other hand `VARCHAR` only uses 1 byte + the content size.

`VARCHAR` requires a prefix value of 1 byte to store the length of the string if the size is less than 256, otherwise it will uses 2 bytes.

One tip is not to use CHAR if you are not going to use all the characters almost all the time, because the size can pile up with empty strings column.

## Character Set

The `UTF8` character set uses a maximum of 3 bytes per character and only contains Basic Multilingual Plane (BMP) characters, which is the home of 65,536 characters (16 bits) from `U+0000` to `U+FFFF`.

The `UTF8mb4` character set uses a maximum of 4 bytes per character including all of BMP characters and Supplementary Multilingual Plane (SMP) which include another possibility of 65,536 new characters from `U+10000` to `U+1FFFF`.

## Emojis (Unicode characters)

`UTF8` can support emojis, but not all of them. All of the new emojis are part of the SMP, so in order to support both basic and supplementary multilingual plane `UTF8mb4` must be used.

The sparkle emoji (✨ `U+2728`) value is between `U+0000` and `U+FFFF` then it can be used on `utf8` charset, but the Woman Health Worker (👩 `U+1F469`) value which is not between `U+0000` and `U+FFFF`, must use the `utf8mb4` charset that range between `U+10000` and `U+1FFFF`.

## Index length

Now after using `utf8mb4` all the characters use 4 bytes instead of 3, so all columns that has more than 191 characters now uses more than 767 bytes, because 192 x 4 bytes is 768 bytes.

Keep in mind the 768 bytes limit is only when using innodb engine and `innodb_large_prefix` is disabled. Since MySQL 5.7 `innodb_large_prefix` is enabled by default allowing up to 3072 bytes. MySIAM has a maximum length of 1000 bytes.

Engine   | Limit
-------- | --------
InnodB with `innodb_large_prefix` disabled   | 768 bytes
MySAIM   | 1000 bytes
InnodB with `innodb_large_prefix` enabled | 3072 bytes

## Solutions

The solve this will depend on what we actually need it can be either removing the index, keep using `utf8`, add a length to the index key or reduce the length of the column.

### Reduce length

For us removing the index wasn't a good option, neither keep using the `utf8`. Reducing the length was possible because the columns will probably never met the actual length which is 255 characters, reducing it to 191 was optimal and in no way impact the table.

### Index length

If changing the length was not possible or desired option, changing the column index to only a chunk of n characters, is another possible option.

```sql
CREATE INDEX `index_name` ON `posts` (title(191));
```
