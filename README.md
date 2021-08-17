# bl-19-books-web
Web front and back end code for the BL books


# LexiDB schema

The database will be called bl-books.

``` json
{
    "name": "data",
    "sets": [
        {
            "name": "tokens",
            "columns" : [
                {
                    "name": "token"
                }
            ]
        },
        {
            "name": "pos",
            "columns": [
                {
                    "name": "pos"
                }
            ]
        },
        {
            "name": "page",
            "rle": true,
            "columns" : [
                {
                    "name": "page"
                }
            ]
        },
        {
            "name": "quality",
            "rle": true,
            "columns" : [
                {
                    "name": "quality",
                    "xml": "quality/@value"
                }
            ]
        },
        {
            "name": "token-count",
            "rle": true,
            "columns" : [
                {
                    "name": "identifier",
                    "xml": "book/@identifier"
                }
            ]
        },
        {
            "name": "file",
            "rle": true,
            "columns": [
                {
                    "name": "$file"
                }
            ]
        }
    ]
}
```

**Note** each time you run this, the data will be formatted to LexiDB format each time, thus duplicating the data you add each time, unless the data you are adding is different.
``` bash
docker run --user $(id -u):$(id -g) -v $(pwd)/test_files:/data --entrypoint "java" --rm ghcr.io/ucrel/lexidb:0.1.1 -Dlog4j.configuration=file:/data/log4j.properties -cp lexidb-2.0.jar util/Insert /data/app.properties bl-books /data/setup_files/.conf.json /data/setup_files
```

At the moment it would appear read permissions is not enough for LexiDB with regards to database files. On the SSL front for `default.conf` have a look at this for the proxy server: https://docs.nginx.com/nginx/admin-guide/security-controls/securing-tcp-traffic-upstream/