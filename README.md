# bl-19-books-web

Web front end code for the British Library books project.

## Installation scripts

In all cases we assume your operating system is Ubuntu.

As the production server requires docker, to install docker run the [installation script.](https://github.com/UCREL/installation-scripts/blob/main/install_docker.sh)


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

**NOTE** `pwd` we assume is at the following path: `/mnt/luna/projects`

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

## Run the web application

### Testing

To run the web application using the test files that are within the `./test_files/setup_files` that after being formatted by LexiDB in the previous section should now be in LexiDB format in the following directory: `./test_files/lexi-data`, use the following docker command:

``` bash
USER=$(id -u) docker-compose up
```

The reason for the `USER=$(id -u)` is due to the folder `./test_files/lexi-data` normally only having permissions for the current user, therefore we run the LexiDB docker container as the current user.

### Production

#### Starting / Shutting down the current production machine

This is currently running on the `ucrel-dighum` server which can only be accessed through the VPN, the current web application can be found running at the following URL: [http://ucrel-dighum.lancaster.ac.uk/](http://ucrel-dighum.lancaster.ac.uk/). This application has 24GB of RAM provisioned for it, the VM itself has access to 31GB of RAM.

Currently on the production server it is running in a `tmux` session for the user `moorea`, therefore if you would like to stop the web application run the following:

``` bash
# Assuming you are the user moorea
tmux attach
# You should now see the docker container /web application running, to stop the docker container
ctrl + C
# This should now start to stop the docker container
docker-compose down # This command stop the docker container
# NOTE $(id -u) needs to evaluate to the user ID of moorea
USER=$(id -u) docker-compose -f production.yml up # This will start the docker container / web application up again
```

#### Log files

You can view the log files anytime including when the application is running.

To view the log files of the web application:
``` bash
docker logs bl-19-books-web_web_1
```

To view the log files of the LexiDB database:
```bash
docker logs bl-19-books-web_lexi_1
```

## Website files

**Note** None of the commands below need re-running, it is only here to show how the POS tag and description table that is within the [./website/query-syntax-guide.html](./website/query-syntax-guide.html) file was generated.

**Note** Python requirements, to run the commands below you will need Python >= 3.6 and install the following pip packages:

``` bash
pip install -r requirements.txt
```

To generate the POS tag and description table that is within the [./website/query-syntax-guide.html](./website/query-syntax-guide.html) file, run the following:

``` bash
python generate_pos_html_tag_table.py > table.html
```

And then copy the contents of [./table.html](./table.html) into [./website/query-syntax-guide.html](./website/query-syntax-guide.html).
