---
title: Droonga plugin development tutorial
layout: en
---

!!WORK IN PROGRESS!!

* TOC
{:toc}

## The goal of this tutorial

Learning steps to develop a Droonga plugin by yourself.

## Precondition

* You must complete [tutorial][].


## Directory Structure

Assume that we are going to add `InputAdapterPlugin` to the system built in [tutorial][]. In that tutorial, Groonga engine was placed under `engine` directory.

Plugins need to be placed in an appropriate directory. For example, `InputAdapterPlugin` should be placed under `lib/droonga/plugin/input_adapter/` directory. Let's create the directory:

    # cd engine
    # mkdir -p lib/droonga/plugin/input_adapter

After creating the directory, the directory structure should be like this:

~~~
engine
├── catalog.json
├── fluentd.conf
└── lib
    └── droonga
        └── plugin
            └── input_adapter
~~~


## Create a plugin

Put a plugin code into `input_adapter` directory.

lib/droonga/plugin/input_adapter/example.rb:

~~~ruby
module Droonga
  class ExampleInputAdapterPlugin < Droonga::InputAdapterPlugin
    repository.register("example", self)
  end
end
~~~

This plugin does nothing except registering itself to Droonga.

## Activate plugin with `catalog.json`

You need to update `catalog.json` to activate your plugin.
Insert following at the last part of `catalog.json` in order to make `"input_adapter"` become a key of the top level hash:

catalog.json:

~~~
(snip)
  },
  "input_adapter": {
    "plugins": ["example", "groonga"]
  },
  "output_adapter": {
    "plugins": ["crud", "groonga"]
  },
  "collector": {
    "plugins": ["basic", "search"]
  },
  "distributor": {
    "plugins": ["search", "crud", "groonga", "watch"]
  }
}
~~~

TODO: the [tutorial][] needs to be updated. After tutorial update, explanation above should also be updated.

## Run

Let's Droonga get started. Note that you need to specify `./lib` directory in `RUBYLIB` environment variable in order to make ruby possible to find your plugin.

~~~
RUBYLIB=./lib fluentd --config fluentd.conf
~~~

## Test

In the previous [tutorial][], we have communicated with `fluent-plugin-droonga` via the protocol adapter built with `expres-droonga`.
For plugin development, sending requests directly to `fluent-plugin-droonga` can be more handy way to debug. We use `fluent-cat` command for this purpose.

Doing in this way also help us to understand internal structure of Droonga.

In the [tutorial][], we have used `fluent-cat` to setup database schema and import data. Do you remember? Sending search request can be done in the similar way.

First, create a request as a JSON.

search-columbus.json:

~~~json
{
  "id": "search:0",
  "dataset": "Starbucks",
  "type": "search",
  "replyTo":"localhost:24224/output",
  "body": {
    "queries": {
      "result": {
        "source": "Store",
        "condition": {
          "query": "Columbus",
          "matchTo": "_key"
        },
        "output": {
          "elements": [
            "startTime",
            "elapsedTime",
            "count",
            "attributes",
            "records"
          ],
          "attributes": ["_key"],
          "limit": -1
        }
      }
    }
  }
}
~~~

This is corresponding to the example to search "Columbus" in the [tutorial][]. Note that the request in `express-droonga` is encapsulated in `"body"` element.

`fluent-cat` expects one line per one JSON object. So we need to use `tr` command to remove line breaks before passing the JSON to `fluent-cat`:

    cat search-columbus.json | tr -d "\n" | fluent-cat starbucks.message

This will output something like below to fluentd's log:

    2014-02-03 14:22:54 +0900 output.message: {"inReplyTo":"search:0","statusCode":200,"type":"search.result","body":{"result":{"count":2,"records":[["2 Columbus Ave. - New York NY  (W)"],["Columbus @ 67th - New York NY  (W)"]]}}}

This is the search result.

If you have [jq][] installed, you can use `jq` instead of `tr`:

    jq -c . search-columbus.json | fluent-cat starbucks.message

## Do something in the plugin

The plugin we have created do nothing so far. Let's get the plugin to do some interesting.

First of all, trap `search` request and log it. Update the plugin like below:

lib/droonga/plugin/input_adapter/example.rb:

~~~
module Droonga
  class ExampleInputAdapterPlugin < Droonga::InputAdapterPlugin
    repository.register("example", self)

    command "search" => :adapt_request
    def adapt_request(input_message)
      $log.info "ExampleInputAdapterPlugin", message: input_message
    end
  end
end
~~~

And restart fluentd, then send the request same as the previous. You will see something like below fluentd's log:

~~~
2014-02-03 16:56:27 +0900 [info]: ExampleInputAdapterPlugin message=#<Droonga::InputMessage:0x007ff36a38cb28 @raw_message={"body"=>{"queries"=>{"result"=>{"output"=>{"limit"=>-1, "attributes"=>["_key"], "elements"=>["startTime", "elapsedTime", "count", "attributes", "records"]}, "condition"=>{"matchTo"=>"_key", "query"=>"Columbus"}, "source"=>"Store"}}}, "replyTo"=>{"type"=>"search.result", "to"=>"localhost:24224/output"}, "type"=>"search", "dataset"=>"Starbucks", "id"=>"search"}>
2014-02-03 16:56:27 +0900 output.message: {"inReplyTo":"search","statusCode":200,"type":"search.result","body":{"result":{"count":2,"records":[["2 Columbus Ave. - New York NY  (W)"],["Columbus @ 67th - New York NY  (W)"]]}}}
~~~

This shows the message is received by our `ExampleInputAdapterPlugin` and then passed to Droonga. Here we can modify the message before the actual data processing.

Suppose that we want to restrict the number of records returned in the response, say `1`. What we need to do is set `limit` to be `1` for every request. Update plugin like below:

lib/droonga/plugin/input_adapter/example.rb:

~~~
module Droonga
  class ExampleInputAdapterPlugin < Droonga::InputAdapterPlugin
    repository.register("example", self)

    command "search" => :adapt_request
    def adapt_request(input_message)
      $log.info "ExampleInputAdapterPlugin", message: input_message
      input_message.body["queries"]["result"]["output"]["limit"] = 1
    end
  end
end
~~~

And restart fluentd. After restart, the response always includes only one record in `records` section:

~~~
2014-02-03 18:47:54 +0900 [info]: ExampleInputAdapterPlugin message=#<Droonga::InputMessage:0x007f913ca6e918 @raw_message={"body"=>{"queries"=>{"result"=>{"output"=>{"limit"=>-1, "attributes"=>["_key"], "elements"=>["startTime", "elapsedTime", "count", "attributes", "records"]}, "condition"=>{"matchTo"=>"_key", "query"=>"Columbus"}, "source"=>"Store"}}}, "replyTo"=>{"type"=>"search.result", "to"=>"localhost:24224/output"}, "type"=>"search", "dataset"=>"Starbucks", "id"=>"search"}>
2014-02-03 18:47:54 +0900 output.message: {"inReplyTo":"search","statusCode":200,"type":"search.result","body":{"result":{"count":2,"records":[["2 Columbus Ave. - New York NY  (W)"]]}}}
~~~

Note that `count` is still `2` because `limit` does not affect `count`. See [search][] for details of `search` command.

  [tutorial]: ../../
  [overview]: ../../../overview/
  [jq]: http://stedolan.github.io/jq/
  [search]: ../../../reference/commands/select/
