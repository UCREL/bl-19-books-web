# bl-19-books-web
Web front and back end code for the BL books


# LexiDB schema

The database will be called bl-books.

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

**Note** each time you run this, the data will be formatted to LexiDB format each time, thus duplicating the data you add each time, unless the data you are adding is different.
``` bash
docker run --user $(id -u):$(id -g) -v $(pwd)/test_files:/data --entrypoint "java" --rm ghcr.io/ucrel/lexidb:0.1.1 -Dlog4j.configuration=file:/data/log4j.properties -cp lexidb-2.0.jar util/Insert /data/app.properties bl-books /data/setup_files/.conf.json /data/setup_files
```

At the moment it would appear read permissions is not enough for LexiDB with regards to database files. On the SSL front for `default.conf` have a look at this for the proxy server: https://docs.nginx.com/nginx/admin-guide/security-controls/securing-tcp-traffic-upstream/



Command used to add data to the lexi database on the server:

docker run --user $(id -u):$(id -g) -v $(pwd)/another:/data --entrypoint "java" --rm ghcr.io/ucrel/lexidb:0.1.1 -Dlog4j.configuration=file:/data/log4j.properties -cp lexidb-2.0.jar util/Insert /data/app.properties bl-books /data/.conf.json /data/alt_1890_english_books_spacy_output

``` bash
docker run --user $(id -u):$(id -g) -v $(pwd)/lexi_1890:/data --entrypoint "java" --rm ghcr.io/ucrel/lexidb:0.1.1 -Dlog4j.configuration=file:/data/log4j.properties -cp lexidb-2.0.jar util/Insert /data/app.properties bl-books /data/.conf.json /data/alt_1890_english_books_spacy_output
```

It took around 3 hours and 13 minutes to format 16GB of data into LexiDB, this was around 9211 volumes of data. In the LexiDB format the data is around 9GB compared to 16GB when in pure JSON format. On my own computer with an SSD it took 4 hours and 16 minutes, I think the task is CPU bound as it only used one processor.



## Web interface:

1. The async function is really good, and is the way forward with respect to giving live data to the user. 
    * However it cannot handle the user going to the next page. I wonder if this can be overcome by using a Framework like [reactjs](https://reactjs.org/) which will only update parts of the frontend that need changing rather than the whole of the frontend (this needs investigating). I think this is required as with the async requests the front page changes each time there is an update, would be good if possible with the sorting to stream the output from LexiDB, this might require a change in the LexiDB API.
    * I'm not sure if it can tell the user that it cannot view the results until all of the results have come in for the sorted searches, if that is how sorted searches work.
    * The way I think async works at the moment is that each time you want an update you do another live search on LexiDB therefore I believe you keep using up threads doing the same thing. This I think is right, I think the threads do stop in the end, but this is not working on my local computer at the moment, on the server it is ok.
2. Rather than running it purely as a frontend application, I think it would be better to run it as a pure web application with a full web framework like [Django](https://www.djangoproject.com/), [flask](https://flask.palletsprojects.com/en/2.0.x/), or [nextjs](https://nextjs.org/). The advantages of having such a framework, it will allow the web server (nginx) to be separate from all of the database within docker (plus for security). It will also allow us to use any type of database engine e.g. Mongo for the meta data. Finally I web framework will allow us to hide all of the database functionality of LexiDB from all users e.g. stop uses from calling LexiDB through the web server (nginx) without having to use the web interface.
3. There might be an issue with the LexiDB API when LexiDB has an exception and therefore fails on the query, need to look into LexiDB more.
4. If using `async=false` when calling the LexiDB API I think we need to call a function within the API to get an update on the progress of the query call.

Benchmarking with 893 million words.

Server using 10 million word blocks:

1. {"token": "the"} -- 38 seconds.
2. {"pos":"NN"}{"pos":"JJ"} - over a minute around 1 minute 15 seconds.
3. This typically times out when you want to know the number of words in the corpus.

Server using 30 million word blocks:

1. {"token": "the"} -- 4 seconds.
2. {"pos":"NN"}{"pos":"JJ"} - around 20 seconds.

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