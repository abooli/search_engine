# HW: Search Engine

**Due date:** Wednesday 27 April (no collaboration extension possible)

In this assignment you will create a highly scalable web search engine.

**Learning Objectives:**
1. Learn to work with a moderate large software project
1. Learn to parallelize data analysis work off the database
1. Learn to work with WARC files and the multi-petabyte common crawl dataset
1. Increase familiarity with indexes and rollup tables for speeding up queries
1. Learn to debug database performance problems

## Task 0: project setup

1. Fork this github repo, and clone your fork onto the lambda server

1. Ensure that you'll have enough free disk space by:
    1. delete the contents of the databases in your `$HOME/bigdata` folder by
        1. see the instructions in the `twitter_postgres_indexes` repo for deleting the contents of the folders created in those assignments
    1. delete your named volumes and previously built docker images by
        1. bringing down any running docker containers
        1. running the command
           ```
           $ docker system prune -a
           ```

## Task 1: getting the system running

In this first task, you will bring up all the docker containers and verify that everything works.

There are three docker-compose files in this repo:
1. `docker-compose.yml` defines the database and pg_bouncer services
1. `docker-compose.override.yml` defines the development flask web app
1. `docker-compose.prod.yml` defines the production flask web app served by nginx

Your tasks are to:

1. Run the script `scripts/create_passwords.sh` to generate the file `.env.prod` containing production credentials for the database.
    Recall that this file is sensitive and should not be added to your git repo for any reason.

1. Modify the `docker-compose.override.yml` file so that the port exposed by the flask service is the same as your userid.

1. Build and bring up the docker containers by running the commands
    ```
    $ docker-compose build
    $ docker-compose up -d
    ```
    Note that the containers have many dependencies,
    and so building them the first time can take an hour or more.
    Further recall that when the `-f` command line flag is not specified, then `docker-compose` will use both the `docker-compose.yml` and `docker-compose.override.yml` configuration files, but will not use the `docker-compose.prod.yml` configuration file.
    For the purposes of this assignment, you won't need to use the production file.

1. Verify that you can connect to your database by running the command:
    ```
    $ docker-compose exec pg psql --user=novichenko
    ```

    > **Historical Note:**
    > Notice that the database username (as defined in the `.prod.env` file) is `novichenko` and not `postgres` or `root`.
    > [Novichenko](https://en.wikipedia.org/wiki/One_Second_for_a_Feat) was a Soviet military officer who saved Kim Il Sung from a grenade assassination attempt.

1. Enable ssh port forwarding so that your local computer can connect to the running flask app.

1. Use firefox on your local computer to connect to the running flask webpage.
   If you've done the previous steps correctly,
   all the buttons on the webpage should work without giving you any error messages,
   but there won't be any data displayed when you search.

1. Edit the script `scripts/check_web_endpoints.sh` so that the port that the script connects to matches the port that the flask server is exposed to on the `docker-compose.overrider.yml` file.

   Then, run the script
   ```
   $ sh scripts/check_web_endpoints.sh
   ```
   to perform automated integration checks that the system is running correctly.
   All tests should report `[pass]`.

## Task 2: loading data

There are two services for loading data:
1. `downloader_warc` loads an entire WARC file into the database; typically, this will be about 100,000 urls from many different hosts. 
1. `downloader_host` searches the all WARC entries in either the common crawl or internet archive that match a particular pattern, and adds all of them into the database

### Task 2a

We'll start with the `downloader_warc` service.
There are two important files in this service:
1. `services/downloader_warc/downloader_warc.py` contains the python code that actually does the insertion
1. `downloader_warc.sh` is a bash script that starts up a new docker container connected to the database, then runs the `downloader_warc.py` file inside that container

Next follow these steps:
1. Visit <https://commoncrawl.org/the-data/get-started/>
1. Find the url of a WARC file.
   On the common crawl website, the paths to WARC files are referenced from the Amazon S3 bucket.
   In order to get a valid HTTP url, you'll need to prepend `https://data.commoncrawl.org/` to the front of the path.
1. Then, run the command
   ```
   $ ./downloader_warc.sh $URL
   ```
   where `$URL` is the url to your selected WARC file.

   This command will spawn a docker container that downloads the WARC file and inserts it into the database.
   Run the command
   ```
   $ docker ps
   ```
   to verify that the docker container is running.
   Since it is downloading and processing a 1GB file, this container will run for a long time.
   Once the WARC file is fully downloaded, the container will automatically stop itself.

   > **Note:**
   > The first time this script is run, a docker image is built.
   > Building this image takes a long time (potentially hours), and so the first run of this script will be slow.
   > Subsequent runs do not have to rebuild the container from scratch and will be much faster.

1. Repeat these steps to download at least 5 different WARC files, each from different years.
   Each of these downloads will spawn its own docker container and can happen in parallel.

You can verify that your system is working with the following tasks.
(Note that they are listed in order of how soon you will start seeing results for them.)
1. Running `docker logs` on your `downloader_warc` containers.
1. Run the query
   ```
   SELECT count(*) FROM metahtml;
   ```
   in psql.
1. Visit your webpage in firefox and verify that search terms are now getting returned.

### Task 2b

The `downloader_warc` service above downloads many urls quickly, but they are mostly low-quality urls.
For example, most URLs do not include the date they were published, and so their contents will not be reflected in the ngrams graph.
In this task, you will implement and run the `downloader_host` service for downloading high quality urls.

1. The file `services/downloader_host/downloader_host.py` has 3 `FIXME` statements.
   You will have to complete the code in these statements to make the python script correctly insert WARC records into the database.

   > **HINT:**
   > The code will require that you use functions from the cdx_toolkit library.
   > You can find the documentation [here](https://pypi.org/project/cdx-toolkit/).
   > You can also reference the `downloader_warc` service for hints,
   > since this service accomplishes a similar task.

1. Run the query
   ```
   SELECT * FROM metahtml_test_summary_host;
   ```
   to display all of the hosts for which the metahtml library has test cases proving it is able to extract publication dates.
   Note that the command above lists the hosts in key syntax form, and you'll have to convert the host into standard form.
1. Select 5 hostnames from the list above, then run the command
   ```
   $ ./downloader_host.sh "$HOST"
   ```
   to insert the urls from these 5 hostnames.

## Task 3: speeding up the webpage

Every time you run a web search on this website, several sql queries are run.
In particular, the full text search is run using a query that contains something like
```
to_tsvector() @@ to_tsquery()
```
in its where clause.
There is currently no index to speed up this query.
So the query will use a sequential scan and the runtime will be linear in the amount of data searched.

That's bad!

Your goal in this task is to create an index that speeds up the query.
It should be a RUM index to speed up the `@@` operator and take advantage of a `LIMIT` clause using an index scan.
The RUM index is already installed on this pg instance,
and that's one of the reason building the images took a long time.

The problem is that I'm not going to tell you what the query is that you need to speed up.
And there's a LOT of code to try to search through... so it's impractical to find the python that causes this query.

Instead, we'll use postgres to tell us what queries are slow without needing to search through the code.
Postgres maintains a relation called `pg_stat_statements` which records the performance of all queries that get run.
You can find a tutorial for using this relation at <https://www.cybertec-postgresql.com/en/postgresql-detecting-slow-queries-quickly/>.

Running the following query in psql will give you the most expensive queries that have been run on the database:
```
SELECT query,
      calls,
      round(total_exec_time::numeric, 2) AS total_time,
      round(mean_exec_time::numeric, 2) AS mean_time,
      round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS percentage
FROM pg_stat_statements
ORDER BY total_exec_time DESC;
```
If you run this query, you'll see that it contains MANY results inside of it because there are many queries that have been run on the database.

In order to find our text search query, modify the `SELECT` statement above to add a `WHERE` clause that requires that the query contain the `@@` symbol.
Now you should see as the first result the `SELECT` query that does full text search on your webpage.
You can verify this by running a few more queries on the webpage and checking that the `calls` column in the `SELECT` query goes up.

Now that you know what the query looks like, write a RUM index to speed up this query.
(One fact that you need which may not be included in the results of your `SELECT` query is that the language parameter to `to_tsquery` is `'simple'`.)

Add this RUM index to your `services/pg/sql/schema.sql` file and run it directly in psql.
You should notice when you run a search, the runtimes of the searches are now faster.

<!--
Since I've given out a lot of work at this point, I don't want to overwhelm you, and I've done this step for you.

There are two steps:
1. create indexes for the fast text search
1. create materialized views for the `count(*)` queries
-->

## Submission

1. Edit this README file to contain the RUM query you created above right here:
    ```
    CREATE INDEX on metahtml using rum(content);
    ```

1. Edit this README file with the results of the following queries in psql.
   The results of these queries will be used to determine if you've completed the previous steps correctly.

    1. This query shows the total number of webpages loaded:
       ```
       select count(*) from metahtml;

       count
       --------
       229392
       (1 row)
       ```

    2. This query shows the number of webpages loaded / hour:
       ```
       select * from metahtml_rollup_insert order by insert_hour desc limit 100;
       hll_count |  url   | hostpathquery | hostpath |  host  |      insert_hour
       -----------+--------+---------------+----------+-------- +------------------------
               5 | 190761 |        187594 |   181054 | 122970 | 2022-05-11 19:00:00+00
               5 |  38890 |         38669 |    36315 |  27928 | 2022-05-11 18:00:00+00
       (2 rows)
       ```

    3. This query shows the hostnames that you have downloaded the most webpages from:
       ```
       select * from metahtml_rollup_host order by hostpath desc limit 100;
         url | hostpathquery | hostpath |                 host
         -----+---------------+----------+--------------------------------------
         68 |            68 |       68 | com,fliphtml5)
         37 |            37 |       37 | fi,fonecta)
         32 |            32 |       32 | edu,virginia,med)
         30 |            30 |       30 | eu,europa,europarl,multimedia)
         21 |            21 |       21 | ru,yandex,realty)
         13 |            13 |       13 | org,momscleanairforce)
         13 |            13 |       13 | com,scribd)
         13 |            13 |       13 | blog,livedoor,xresolutionx)
         13 |            13 |       13 | ru,gameguru)
         13 |            13 |       13 | com,motherjones,preprod)
         13 |            13 |       13 | com,microsoft,azuremarketplace)
         13 |            13 |       13 | com,apple,music)
         13 |            13 |       13 | edu,harvard,wiki)
         12 |            12 |       12 | com,shazam)
         12 |            12 |       12 | gr,news247)
         12 |            12 |       12 | cz,pravednes)
         12 |            12 |       12 | pt,vodafone,manuai   17 |       17 | ch,tagblatt)
         16 |            16 |       16 | gov,recreation)
         16 |            16 |       16 | com,eplane)
         16 |            16 |       16 | com,hollandandbarrett)
         15 |            15 |       15 | eu,europa,echa)
         14 |            14 |       14 | com,adobe,experienceleague)
         14 |            14 |       14 | com,abc15)
         14 |            14 |       14 | net,advertigo)
         13 |            13 |       13 | org,momscleanairforce)
         13 |            13 |       13 | com,scribd)
         13 |            13 |       13 | blog,livedoor,xresolutionx)
         13 |            13 |       13 | ru,gameguru)
         13 |            13 |       13 | com,motherjones,preprod)
         13 |            13 |       13 | com,microsoft,azuremarketplace)
         13 |            13 |       13 | com,apple,music)
         13 |            13 |       13 | edu,harvard,wiki)
         12 |            12 |       12 | com,shazam)
         12 |            12 |       12 | gr,news247)
         12 |            12 |       12 | cz,pravednes)
         12 |            12 |       12 | pt,vodafone,manuaisonline)
         12 |            12 |       12 | com,magnoliabox)
         12 |            12 |       12 | com,juridicas,noticias)
         12 |            12 |       12 | nl,npo3)
         12 |            12 |       12 | tw,com,pchome,stock)
         12 |            12 |       12 | com,flipp)
         12 |            12 |       12 | edu,kit,hoc,studium)
         12 |            12 |       12 | com,github)
         12 |            12 |       12 | ca,macleans)
         11 |            11 |       11 | uk,co,independent)
         11 |            11 |       11 | au,com,museumsvictoria,collections)
         11 |            11 |       11 | ru,5-tv)
         11 |            11 |       11 | com,bigstockphoto)
         17 |            17 |       11 | org,fao,agrovoc)
         11 |            11 |       11 | com,imdb)
         11 |            11 |       11 | com,tcm,prod-www)
         11 |            11 |       11 | com,fastcompany,fc-resources)
         11 |            11 |       11 | org,maplight,classic)
         11 |            11 |       11 | de,utrace,en)
         11 |            11 |       11 | org,wikipedia,sl)
         11 |            11 |       11 | com,intereconomia)
         11 |            11 |       11 | com,discountmags)
         10 |            10 |       10 | net,slideshare)
         10 |            10 |       10 | vn,soha,tratu)
         13 |            13 |       10 | com,yahoo,buy,tw)
         10 |            10 |       10 | com,foxnews)
         10 |            10 |       10 | de,focus)
         10 |            10 |       10 | com,elsevier,pure,minnesota-staging)
         10 |            10 |       10 | com,sportslens)
         10 |            10 |       10 | com,donga,sports)
         10 |            10 |       10 | com,soundcloud)
         10 |            10 |       10 | com,roche,streamingmedia)
         10 |            10 |       10 | com,stomt)
         10 |            10 |       10 | pl,spidersweb)
         10 |            10 |       10 | com,google,docs)
         10 |            10 |       10 | info,moondo,mangiare)
         10 |            10 |       10 | jp,co,yahoo,lohaco)
         10 |            10 |       10 | com,globedia,mx)
         15 |            15 |       10 | com,hp)
         10 |            10 |       10 | it,gazzetta,store)
         10 |            10 |       10 | tv,gloria)
         10 |            10 |       10 | it,messinaora)
         10 |            10 |       10 | ru,pics)
         10 |            10 |       10 | com,microsoft,docs)
         10 |            10 |       10 | com,amyscottphotography)
         10 |            10 |       10 | nl,2miljoen)
         10 |            10 |       10 | com,olympics)
         10 |            10 |       10 | com,123rf,de)
          9 |             9 |        9 | de,gewusst-wo)
          9 |             9 |        9 | pl,rynekzdrowia)
          9 |             9 |        9 | jp,sakenomy)
          9 |             9 |        9 | com,tradingsat)
          9 |             9 |        9 | org,gnbdc)
          9 |             9 |        9 | nl,graftombe)
          9 |             9 |        9 | com,wsj,cn)
          9 |             9 |        9 | com,secondlife,maps)
          9 |             9 |        9 | com,haribhoomi)
          9 |             9 |        9 | ru,org,linux)
          9 |             9 |        9 | com,tribuna,ua)
         11 |            11 |        9 | org,un,unstats)
          9 |             9 |        9 | com,steampowered,help)
          9 |             9 |        9 | edu,ucsd,can,supersearch)
          9 |             9 |        9 | edu,ncsu,lib,staff)
          9 |             9 |        9 | jp,co,p1mode)
         10 |            10 |        9 | com,etsy)
         (100 rows)
       ```

1. Take a screenshot of an interesting search result.
   Ensure that the timer on the bottom of the webpage is included in the screenshot.
   Add the screenshot to your git repo, and modify the `<img>` tag below to point to the screenshot.

   <img src='screenshot.png' />

1. Commit and push your changes to github.

1. Submit the link to your github repo in sakai.
