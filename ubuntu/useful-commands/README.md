# Useful commands

* [1. Utility](#1-utility)
    * [1. Create random 64-character alphanumeric string](#1-create-random-64-character-alphanumeric-string)

## 1. Utility

#### 1. Create random 64-character alphanumeric string 
```shell script
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1
```
**NOTE**: You can replace the `64` with any number to create any character length string.
