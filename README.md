# bl-19-books-web

Web front end code for the British Library books project.


## LexiDB Formatting and Schema

### Schema

``` json
{
    "name": "tokens",
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

### Formatting

This is to get it into a format that LexiDB, this formatting process takes a TSV and converts it into the file system LexiDB uses.

**Note** each time you run a insert command, the data will be formatted to LexiDB format each time, thus duplicating the data you add each time, unless the data you are adding is different.

In both cases the database will be called `bl-books`.

#### Testing

``` bash
docker run --user $(id -u):$(id -g) -v $(pwd)/test_files:/data --entrypoint "java" --rm ghcr.io/ucrel/lexidb:0.1.1 -Dlog4j.configuration=file:/data/log4j.properties -cp lexidb-2.0.jar util/Insert /data/app.properties bl-books /data/setup_files/.conf.json /data/setup_files
```

This command will run as the current user `--user $(id -u):$(id -g)`, it will use [./test_files/log4j.properties](./test_files/log4j.properties) as the [log4j properties file](https://docs.oracle.com/cd/E29578_01/webhelp/cas_webcrawler/src/cwcg_config_log4j_file.html) which in this case tell it to write to the log file `log4j.appender.FILE.File=/tmp/lexidb.log`. For more details on the rest of the command see the [LexiDB formatting / Importing data guide](https://github.com/UCREL/lexidb#formatting--importing-data). 

**Note** the license associated with the test files within [./test_files/setup_files](./test_files/setup_files) is [CC Public Domain Mark 1.0](https://creativecommons.org/publicdomain/mark/1.0/) as they have come from the [British Library 19th century book corpus.](https://doi.org/10.21250/db14)


#### Production

Command used to add data to the LexiDB database on the server:

``` bash
docker run --user $(id -u):$(id -g) -v $(pwd)/lexi_1890:/data --entrypoint "java" --rm ghcr.io/ucrel/lexidb:0.1.1 -Dlog4j.configuration=file:/data/log4j.properties -cp lexidb-2.0.jar util/Insert /data/app.properties bl-books /data/.conf.json /data/alt_1890_english_books_spacy_output
```

After running this command the formatted data will be at the following file path:

``` bash
$(pwd)/lexi_1890/large-lexi-data
```

For this production version to increase the query speed we changed the default `app.properties` with the following changes (note the `data.path` changes was only changed as we had been experimenting with different block sizes).

```
data.path=/data/large-lexi-data
block.size=30000000
```

The trade off with having a larger `block.size` is that you need more RAM.

It took around 3 hours and 13 minutes to format 16GB of data into LexiDB, this was around 9211 volumes of data. In the LexiDB format the data is around 7.8GB compared to 16GB when in pure JSON format.

## Web interface:

1. The async function is really good, and is the way forward with respect to giving live data to the user. 
    * However it cannot handle the user going to the next page. I wonder if this can be overcome by using a Framework like [reactjs](https://reactjs.org/) which will only update parts of the frontend that need changing rather than the whole of the frontend (this needs investigating). I think this is required as with the async requests the front page changes each time there is an update, would be good if possible with the sorting to stream the output from LexiDB, this might require a change in the LexiDB API.
    * I'm not sure if it can tell the user that it cannot view the results until all of the results have come in for the sorted searches, if that is how sorted searches work.
    * The way I think async works at the moment is that each time you want an update you do another live search on LexiDB therefore I believe you keep using up threads doing the same thing. This I think is right, I think the threads do stop in the end, but this is not working on my local computer at the moment, on the server it is ok.
2. Rather than running it purely as a frontend application, I think it would be better to run it as a pure web application with a full web framework like [Django](https://www.djangoproject.com/), [flask](https://flask.palletsprojects.com/en/2.0.x/), or [nextjs](https://nextjs.org/). The advantages of having such a framework, it will allow the web server (nginx) to be separate from all of the database within docker (plus for security). It will also allow us to use any type of database engine e.g. Mongo for the meta data. Finally I web framework will allow us to hide all of the database functionality of LexiDB from all users e.g. stop uses from calling LexiDB through the web server (nginx) without having to use the web interface.
3. There might be an issue with the LexiDB API when LexiDB has an exception and therefore fails on the query, need to look into LexiDB more.
4. If using `async=false` when calling the LexiDB API I think we need to call a function within the API to get an update on the progress of the query call.
5. At the moment it would appear read permissions is not enough for LexiDB with regards to database files. On the SSL front for `default.conf` have a look at this for the proxy server: https://docs.nginx.com/nginx/admin-guide/security-controls/securing-tcp-traffic-upstream/
6. LexiDB performs better with more RAM, the disk type does not seem to affect the performance that much from my own experience.

Benchmarking with 893 million words.

Server using 10 million word blocks:

1. {"token": "the"} -- 38 seconds.
2. {"pos":"NN"}{"pos":"JJ"} - over a minute around 1 minute 15 seconds.
3. This typically times out when you want to know the number of words in the corpus.

Server using 30 million word blocks:

1. {"token": "the"} -- 4 seconds.
2. {"pos":"NN"}{"pos":"JJ"} - around 13 seconds.

Server tried using 100 million word blocks on inserting into lexiDB and it failed with a memory error.

Server using 30 million word blocks and a SSD rather than luna.

1. {"token": "the"} -- 4 seconds.
2. {"pos":"NN"}{"pos":"JJ"} - 10 seconds.



## Website files

To generate the POS tag and description table that is within the [./website/query-syntax-guide.html](./website/query-syntax-guide.html) file, run the following:

``` bash
python generate_pos_html_tag_table.py > table.html
```

And then copy the contents of [./table.html](./table.html) into [./website/query-syntax-guide.html](./website/query-syntax-guide.html).