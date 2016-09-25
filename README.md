# elastic-match

## A Generic Restful Match Engine

link:"http://elasticmatch.com":http://elasticmatch.com

#### elastic-match is a distributed RESTful matching engine built for the cloud. Features include:

* Distributed and Highly Available Matching Engine.
** Each index is fully sharded with a configurable number of shards.
** Each shard can have one or more replicas.
** Read / Search operations performed on any of the replica shards.
* Multi Tenant with Multi Types.
** Support for more than one index.
** Support for more than one type per index.
** Index level configuration (number of shards, index storage, ...).
* Various set of APIs
** HTTP RESTful API
** Native Java API.
** All APIs perform automatic node operation rerouting.
* Document oriented
** No need for upfront schema definition.
** Schema can be defined per type for customization of the indexing process.
* Reliable, Asynchronous Write Behind for long term persistency.
* (Near) Real Time Search.
* Built on top of Lucene
** Each shard is a fully functional Lucene index
** All the power of Lucene easily exposed through simple configuration / plugins.
* Per operation consistency
** Single document level operations are atomic, consistent, isolated and durable.
* Open Source under the Apache License, version 2 ("ALv2")

h2. Getting Started

First of all, DON'T PANIC. It will take 5 minutes to get the gist of what Elasticsearch is all about.

h3. Requirements

You need to have a recent version of Java installed. See the "Setup":http://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html#jvm-version page for more information.

h3. Installation

* "Download":https://www.elastic.co/downloads/elasticsearch and unzip the Elasticsearch official distribution.
* Run @bin/elasticsearch@ on unix, or @bin\elasticsearch.bat@ on windows.
* Run @curl -X GET http://localhost:9200/@.
* Start more servers ...

h3. Indexing

Let's try and index some twitter like information. First, let's create a twitter user, and add some tweets (the @twitter@ index will be created automatically):

<pre>
curl -XPUT 'http://localhost:9200/twitter/user/kimchy?pretty' -d '{ "name" : "Shay Banon" }'

curl -XPUT 'http://localhost:9200/twitter/tweet/1?pretty' -d '
{
    "user": "kimchy",
    "post_date": "2009-11-15T13:12:00",
    "message": "Trying out Elasticsearch, so far so good?"
}'

curl -XPUT 'http://localhost:9200/twitter/tweet/2?pretty' -d '
{
    "user": "kimchy",
    "post_date": "2009-11-15T14:12:12",
    "message": "Another tweet, will it be indexed?"
}'
</pre>

Now, let's see if the information was added by GETting it:

<pre>
curl -XGET 'http://localhost:9200/twitter/user/kimchy?pretty=true'
curl -XGET 'http://localhost:9200/twitter/tweet/1?pretty=true'
curl -XGET 'http://localhost:9200/twitter/tweet/2?pretty=true'
</pre>

h3. Searching

Mmm search..., shouldn't it be elastic?
Let's find all the tweets that @kimchy@ posted:

<pre>
curl -XGET 'http://localhost:9200/twitter/tweet/_search?q=user:kimchy&pretty=true'
</pre>

We can also use the JSON query language Elasticsearch provides instead of a query string:

<pre>
curl -XGET 'http://localhost:9200/twitter/tweet/_search?pretty=true' -d '
{
    "query" : {
        "match" : { "user": "kimchy" }
    }
}'
</pre>

Just for kicks, let's get all the documents stored (we should see the user as well):

<pre>
curl -XGET 'http://localhost:9200/twitter/_search?pretty=true' -d '
{
    "query" : {
        "match_all" : {}
    }
}'
</pre>

We can also do range search (the @postDate@ was automatically identified as date)

<pre>
curl -XGET 'http://localhost:9200/twitter/_search?pretty=true' -d '
{
    "query" : {
        "range" : {
            "post_date" : { "from" : "2009-11-15T13:00:00", "to" : "2009-11-15T14:00:00" }
        }
    }
}'
</pre>

There are many more options to perform search, after all, it's a search product no? All the familiar Lucene queries are available through the JSON query language, or through the query parser.

h3. Multi Tenant - Indices and Types

Maan, that twitter index might get big (in this case, index size == valuation). Let's see if we can structure our twitter system a bit differently in order to support such large amounts of data.

Elasticsearch supports multiple indices, as well as multiple types per index. In the previous example we used an index called @twitter@, with two types, @user@ and @tweet@.

Another way to define our simple twitter system is to have a different index per user (note, though that each index has an overhead). Here is the indexing curl's in this case:

<pre>
curl -XPUT 'http://localhost:9200/kimchy/info/1?pretty' -d '{ "name" : "Shay Banon" }'

curl -XPUT 'http://localhost:9200/kimchy/tweet/1?pretty' -d '
{
    "user": "kimchy",
    "post_date": "2009-11-15T13:12:00",
    "message": "Trying out Elasticsearch, so far so good?"
}'

curl -XPUT 'http://localhost:9200/kimchy/tweet/2?pretty' -d '
{
    "user": "kimchy",
    "post_date": "2009-11-15T14:12:12",
    "message": "Another tweet, will it be indexed?"
}'
</pre>

The above will index information into the @kimchy@ index, with two types, @info@ and @tweet@. Each user will get their own special index.

Complete control on the index level is allowed. As an example, in the above case, we would want to change from the default 5 shards with 1 replica per index, to only 1 shard with 1 replica per index (== per twitter user). Here is how this can be done (the configuration can be in yaml as well):

<pre>
curl -XPUT http://localhost:9200/another_user?pretty -d '
{
    "index" : {
        "number_of_shards" : 1,
        "number_of_replicas" : 1
    }
}'
</pre>

Search (and similar operations) are multi index aware. This means that we can easily search on more than one
index (twitter user), for example:

<pre>
curl -XGET 'http://localhost:9200/kimchy,another_user/_search?pretty=true' -d '
{
    "query" : {
        "match_all" : {}
    }
}'
</pre>

Or on all the indices:

<pre>
curl -XGET 'http://localhost:9200/_search?pretty=true' -d '
{
    "query" : {
        "match_all" : {}
    }
}'
</pre>

{One liner teaser}: And the cool part about that? You can easily search on multiple twitter users (indices), with different boost levels per user (index), making social search so much simpler (results from my friends rank higher than results from friends of my friends).

h3. Distributed, Highly Available

Let's face it, things will fail....

Elasticsearch is a highly available and distributed search engine. Each index is broken down into shards, and each shard can have one or more replica. By default, an index is created with 5 shards and 1 replica per shard (5/1). There are many topologies that can be used, including 1/10 (improve search performance), or 20/1 (improve indexing performance, with search executed in a map reduce fashion across shards).

In order to play with the distributed nature of Elasticsearch, simply bring more nodes up and shut down nodes. The system will continue to serve requests (make sure you use the correct http port) with the latest data indexed.

h3. Where to go from here?

We have just covered a very small portion of what Elasticsearch is all about. For more information, please refer to the "elastic.co":http://www.elastic.co/products/elasticsearch website. General questions can be asked on the "Elastic Discourse forum":https://discuss.elastic.co or on IRC on Freenode at "#elasticsearch":https://webchat.freenode.net/#elasticsearch. The Elasticsearch GitHub repository is reserved for bug reports and feature requests only.

h3. Building from Source

Elasticsearch uses "Gradle":https://gradle.org for its build system. You'll need to have a modern version of Gradle installed - 2.13 should do.

In order to create a distribution, simply run the @gradle assemble@ command in the cloned directory.

The distribution for each project will be created under the @build/distributions@ directory in that project.

See the "TESTING":TESTING.asciidoc file for more information about
running the Elasticsearch test suite.

h3. Upgrading from Elasticsearch 1.x?

In order to ensure a smooth upgrade process from earlier versions of
Elasticsearch (1.x), it is required to perform a full cluster restart. Please
see the "setup reference":
https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html
for more details on the upgrade process.

h1. License

<pre>
This software is licensed under the Apache License, version 2 ("ALv2"), quoted below.

Copyright 2009-2016 Elasticsearch <https://www.elastic.co>

Licensed under the Apache License, Version 2.0 (the "License"); you may not
use this file except in compliance with the License. You may obtain a copy of
the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations under
the License.
</pre>
