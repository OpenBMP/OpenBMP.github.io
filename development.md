---
title: Developer
excerpt: Developer Information
nav_order: 10
nav_exclude: false
search_exclude: false
---
# OpenBMP Development

OpenBMP is developed using a few programming languages with various dependencies.
In effort to ease developers from having to build and install these dependencies,
the [docker dev image](https://github.com/OpenBMP/obmp-docker/tree/main/dev-image)
should be used. 

## Running developer docker image
In general, you can run the docker image using the following:

```
docker run --rm -v $(PWD):/ws -it openbmp/dev-image /bin/bash
```