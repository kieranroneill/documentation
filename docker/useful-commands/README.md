# Useful commands

#### Table of contents

* [1. General](#1-general)
* [2. Containers](#2-containers)
* [3. Images](#3-images)
    * [1. Remove all dangling images](#1-remove-all-dangling-images)

## 1. General

[&#8593; Back to the top](#table-of-contents)

## 2. Containers

[&#8593; Back to the top](#table-of-contents)

## 3. Images

#### 1. Remove all dangling images
```shell script
docker rmi -f $(docker images -f "dangling=true" -q)
```

[&#8593; Back to the top](#table-of-contents)
