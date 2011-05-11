---
layout: blogpost
title: Data Visualization with ElasticSearch and Protovis
cat: blog
author: Karel Minarik
nick: karmiq
---

# Data Visualization with ElasticSearch and Protovis

The primary purpose of a search engine is, quite unsurprisingly: _searching_. You pass it a query, and it returns bunch of matching documents, in the order of relevance. We can get creative with query construction, experimenting with different analyzers for our documents, and the search engine tries hard to provide best results.

Nevertheless, a modern full-text search engine can do much more than that. At its core lies the <a href="http://en.wikipedia.org/wiki/Index_(search_engine)#Inverted_indices"><em>inverted index</em></a>, a highly optimized data structure for efficient lookup of documents matching the query. But it also allows to compute complex **aggregations** of our data, called [_facets_](http://www.elasticsearch.org/guide/reference/api/search/facets/index.html).

The usual purpose of facets is to offer the user a _faceted navigation_, or _faceted search_. When you search for “camera” at an online store, you can refine your search by choosing different manufacturers, price ranges, or features, usually by clicking on a link, not by fiddling with the query syntax.

A canonical example of a [faceted navigation at _LinkedIn_](http://blog.linkedin.com/2009/12/14/linkedin-faceted-search/) is pictured below.

![Faceted Search at LinkedIn](/blog/images/dashboards/linkedin-faceted-search.png)

Faceted search is one of the few ways to make powerful queries accessible to your users: see Moritz Stefaner's experiments with [“Elastic Lists”](http://well-formed-data.net/archives/54/elastic-lists) for inspiration.

But, we can do much more with facets then just displaying these links and checkboxes.
We can use the data for makings **charts**, which is exactly what we'll do in this article.


## Live Dashboards ##

In almost any analytical, monitoring or data-mining service you'll hit the requirement _“We need a dashboard!”_ sooner or later. Because everybody loves dashboards, whether they're useful or just pretty. As it happens, we can use facets as a pretty powerful analytical engine for our data, without writing any [OLAP](http://en.wikipedia.org/wiki/Online_analytical_processing) implementations.

The screenshot below is from a [social media monitoring application](http://ataxosocialinsider.cz/) which uses _ElasticSearch_ not only to search and mine the data, but also to provide data aggregation for the interactive dashboard.

![Ataxo Social Insider Dashboard](/blog/images/dashboards/dashboard.png)

When the user drills down into the data, adds a keyword, uses a custom query, all the charts change in real-time, thanks to the way how facet aggregation works. The dashboard is not a static snapshot of the data, pre-calculated periodically, but a truly interactive tool for data exploration.

In this article, we'll learn how to retrieve data for charts like these from _ElasticSearch_, and how to create the charts themselves.


## Pie charts with a _terms_ facet ##

For the first chart, we'll use a simple [_terms_](http://elasticsearch.org/guide/reference/api/search/facets/terms-facet.html) facet in _ElasticSearch_. This facet returns the most frequent terms for a field, together with occurence counts.

Let's index some example data first.

<pre class="prettyprint lang-bash">
curl -X DELETE "http://localhost:9200/dashboard"
curl -X POST "http://localhost:9200/dashboard/article" -d '
             { "title" : "One",
               "tags"  : ["ruby", "java", "search"]}
'
curl -X POST "http://localhost:9200/dashboard/article" -d '
             { "title" : "Two",
               "tags"  : ["java", "search"] }
'
curl -X POST "http://localhost:9200/dashboard/article" -d '
             { "title" : "Three",
               "tags"  : ["erlang", "search"] }
'
curl -X POST "http://localhost:9200/dashboard/article" -d '
             { "title" : "Four",
               "tags"  : ["search"] }
'
curl -X POST "http://localhost:9200/dashboard/_refresh"
</pre>

As you see, we are storing four articles, each with a couple of tags; an article can have multiple tags, which is trivial to express in _ElasticSearch_'s document format, JSON.

Now, to retrieve “Top Ten Tags” across the documents, we can simply do:

<pre class="prettyprint lang-bash">
curl -X POST "http://localhost:9200/dashboard/_search?pretty=true" -d '
  {
    "query" : { "match_all" : {} },

    "facets" : {
      "tags" : { "terms" : {"field" : "tags", "size" : 10} }
    }
  }
'
</pre>

You can see that we are retrieving all documents, and we have defined a terms facet called “tags”. This query will return something like this:

<pre class="prettyprint lang-js">
{
  "took" : 2,
  // ... snip ...
  "hits" : {
    "total" : 4,
    // ... snip ...
  },
  "facets" : {
    "tags" : {
      "_type" : "terms",
      "missing" : 1,
      "terms" : [ {
        "term" : "search",
        "count" : 4
      }, {
        "term" : "java",
        "count" : 2
      }, {
        "term" : "ruby",
        "count" : 1
      }, {
        "term" : "erlang",
        "count" : 1
      } ]
    }
  }
}
</pre>

We are interested in the `facets` section of the JSON, notably in the `facets.tags.terms` array. It tells us that we have four articles tagged _search_, two tagged _java_, and so on. (Of course, we could add a `size` parameter to the query, to skip the results altogether.)

Suitable visualization for this type of ratio distribution is a pie chart, or its variation: a donut chart. The end result is displayed below (you may want to check out the [working example](/blog/assets/dashboards/donut.html)).

![Donut Chart](/blog/images/dashboards/donut_chart.png)

We will use [_Protovis_](http://vis.stanford.edu/protovis/), a JavaScript data visualization toolkit. _Protovis_ is 100% open source, and you could think of it as _Ruby on Rails_ for data visualization; in stark contrast to similar tools, it does not ship with a limited set of chart types to “choose” from, but it defines a set of primitives and a flexible domain-specific language so you can easily build your own custom visualizations. Creating [pie charts](http://vis.stanford.edu/protovis/ex/pie.html) is pretty easy in _Protovis_.

Since _ElasticSearch_ returns JSON data, we can load it with a simple Ajax call. Don't forget that you can clone or download the [full source code](https://gist.github.com/966338) for this example.

First, we need a HTML file to contain our chart and to load the data from _ElasticSearch_:

<pre class="prettyprint lang-js">
&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;  
  &lt;title&gt;ElasticSearch Terms Facet Donut Chart&lt;/title&gt;
  &lt;meta http-equiv=&quot;Content-Type&quot; content=&quot;text/html; charset=utf-8&quot; /&gt;

  &lt;!-- Load JS libraries --&gt;
  &lt;script src=&quot;jquery-1.5.1.min.js&quot;&gt;&lt;/script&gt;
  &lt;script src=&quot;protovis-r3.2.js&quot;&gt;&lt;/script&gt;
  &lt;script src=&quot;donut.js&quot;&gt;&lt;/script&gt;
  &lt;script&gt;
    $( function() { load_data(); });

    var load_data = function() {
      $.ajax({   url: &#x27;http://localhost:9200/dashboard/article/_search?pretty=true&#x27;
               , type: &#x27;POST&#x27;
               , data :
                  JSON.stringify(
                  {
                    &quot;query&quot; : { &quot;match_all&quot; : {} },

                    &quot;facets&quot; : {
                      &quot;tags&quot; : {
                        &quot;terms&quot; : {
                            &quot;field&quot; : &quot;tags&quot;,
                            &quot;size&quot;  : &quot;10&quot;
                        }
                      }
                    }
                  })
               , dataType : &#x27;json&#x27;
               , processData: false
               , success: function(json, statusText, xhr) {
                   return display_chart(json);
                 }
               , error: function(xhr, message, error) {
                   console.error(&quot;Error while loading data from ElasticSearch&quot;, message);
                   throw(error);
                 }          
      });

      var display_chart = function(json) {
        Donut().data(json.facets.tags.terms).draw();
      };

    };
  &lt;/script&gt;
&lt;/head&gt;
&lt;body&gt;

  &lt;!-- Placeholder for the chart --&gt;
  &lt;div id=&quot;chart&quot;&gt;&lt;/div&gt;

&lt;/body&gt;
&lt;/html&gt;
</pre>

On document load, we retrieve exactly the same facet, via Ajax, as we did earlier with Curl. In the jQuery Ajax _callback_, we pass the returned JSON to the `Donut()` function via the `display_chart()` wrapper.

The `Donut()` function itself is displayed, with annotations, below:

<pre class="prettyprint lang-js">
// ===========================
// A donut chart with Protovis
// ===========================
//
// See http://vis.stanford.edu/protovis/ex/pie.html 
//
var Donut = function(dom_id) {
  // Set the default DOM element ID to bind
  if (&#x27;undefined&#x27; == typeof dom_id)  dom_id = &#x27;chart&#x27;;

  var data = function(json) {
    // Set the data for the chart
    this.data = json;
    return this;
  };

  var draw = function() {

          // Sort the data by term names, so the color scheme for wedges is preserved with any order
      var entries = this.data.sort( function(a, b) { return a.term &lt; b.term ? -1 : 1; } ),
          // Create an array holding just the values (counts)
          values  = pv.map(entries, function(e) { return e.count; });
      // console.log(&#x27;Drawing&#x27;, entries, values);

          // Set-up dimensions and color scheme for the chart
      var w = 200,
          h = 200,
          colors = pv.Colors.category10().range();

          // Create the basis panel
      var vis = new pv.Panel()
          .width(w)
          .height(h)
          .margin(0, 0, 0, 0);

          // Create the &quot;wedges&quot; of the chart
      vis.add(pv.Wedge)
          // Set-up auxiliary variable to hold state (mouse over / out)
          .def(&quot;active&quot;, -1)
          // Pass the normalized data to Protovis
          .data( pv.normalize(values) )
          // Set-up chart position and dimension
          .left(w/3)
          .top(w/3)
          .outerRadius(w/3)
          // Create a &quot;donut hole&quot; in the center of the chart
          .innerRadius(15)
          // Compute the &quot;width&quot; of the wedge
          .angle( function(d) { return d * 2 * Math.PI; } )
          // Add white stroke
          .strokeStyle(&quot;#fff&quot;)

          // On &quot;mouse over&quot;, set the relevant &quot;wedge&quot; as active
          .event(&quot;mouseover&quot;, function() {
             this.active(this.index);
             this.cursor(&#x27;pointer&#x27;);
             return this.root.render();
           })
           // On &quot;mouse out&quot;, clear the active state
           .event(&quot;mouseout&quot;,  function() {
             this.active(-1);
             return this.root.render();
           })
           // On &quot;mouse down&quot;, perform action, such as filtering the results...
           .event(&quot;mousedown&quot;, function(d) {
             var term = entries[this.index].term;
             return (alert(&quot;Filter the results by &#x27;&quot;+term+&quot;&#x27;&quot;));
           })

           // Add the left part of he &quot;inline&quot; label, displayed inside the donut &quot;hole&quot;
           .anchor(&quot;right&quot;).add(pv.Dot)
            // The label is visible when the corresponding &quot;wedge&quot; is active
            .visible( function() { return this.parent.children[0].active() == this.index; } )
            .fillStyle(&quot;#222&quot;)
            .lineWidth(0)
            .radius(14)

           // Add the middle part of the label
           .anchor(&quot;center&quot;).add(pv.Bar)
            .fillStyle(&quot;#222&quot;)
            .width(function(d) {                                // Compute width:
              return (d*100).toFixed(1).toString().length*4 +   // add pixels for percents
                     10 +                                       // add pixels for glyphs (%, etc)
                     (entries[this.index].term.length*9);       // add pixels for letters (very rough)
            })
            .height(28)
            .top((w/3)-14)

           // Add the right part of the label
           .anchor(&quot;right&quot;).add(pv.Dot)
            .fillStyle(&quot;#222&quot;)
            .lineWidth(0)
            .radius(14)

           // Add the text to label
           .parent.children[2].anchor(&quot;left&quot;).add(pv.Label)
            .left((w/3)-7)
            .text(function(d) {
              // Combine the text for label
              return (d*100).toFixed(1) + &quot;%&quot; + &#x27; &#x27; +
                     entries[this.index].term + &#x27; (&#x27; + values[this.index] + &#x27;)&#x27;;
            })
            .textStyle(&quot;#fff&quot;)

          // Bind the chart to DOM element
          .root.canvas(dom_id)
          // And render it.
          .render();
  };

  // Create the public API
  return {
    data   : data,
    draw   : draw
  };

};
</pre>

As you can see, with a simple transformation of JSON data returned from _ElasticSearch_, we're able to create rich, attractive visualization of tag distribution among our articles. You can see the full example [here](/blog/assets/dashboards/donut.html).

It's worth repeating that the visualization will work in _exactly the same way_ when we use a different query, such as displaying only articles written by a certain author or published in certain date range.


## Timelines with a _date histogram_ facets

_Protovis_ makes it very easy to create another common form of visualization: the [_timeline_](http://vis.stanford.edu/protovis/ex/zoom.html). Any type of data, tied to a certain date, such as an article being published, an event taking place, a purchase being completed can be visualized on a timeline.

The end result should look like this (again, you may want to check out the [working version](/blog/assets/dashboards/timeline.html)):

![Timeline Chart](/blog/images/dashboards/timeline_chart.png)

So, let's store handful of articles with a `published` date in the index.

<pre class="prettyprint lang-bash">
curl -X DELETE "http://localhost:9200/dashboard"
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "1",  "published" : "2011-01-01" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "2",  "published" : "2011-01-02" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "3",  "published" : "2011-01-02" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "4",  "published" : "2011-01-03" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "5",  "published" : "2011-01-04" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "6",  "published" : "2011-01-04" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "7",  "published" : "2011-01-04" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "8",  "published" : "2011-01-04" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "9",  "published" : "2011-01-10" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "10", "published" : "2011-01-12" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "11", "published" : "2011-01-13" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "12", "published" : "2011-01-14" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "13", "published" : "2011-01-14" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "14", "published" : "2011-01-15" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "15", "published" : "2011-01-20" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "16", "published" : "2011-01-20" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "17", "published" : "2011-01-21" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "18", "published" : "2011-01-22" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "19", "published" : "2011-01-23" }'
curl -X POST "http://localhost:9200/dashboard/article" -d '{ "t" : "20", "published" : "2011-01-24" }'
curl -X POST "http://localhost:9200/dashboard/_refresh"
</pre>

To retrieve the frequency of articles being published, we'll use a [_date histogram_ facet](http://www.elasticsearch.org/guide/reference/api/search/facets/date-histogram-facet.html) in _ElasticSearch_.

<pre class="prettyprint lang-bash">
curl -X POST "http://localhost:9200/dashboard/_search?pretty=true" -d '
{
  "query" : { "match_all" : {} },

  "facets" : {
    "published_on" : {
      "date_histogram" : {
          "field"    : "published",
          "interval" : "day"
      }
    }
  }
}
'
</pre>

Notice how we set the interval to _day_; we could easily change the granularity of the histogram to _week_, _month_, or _year_.

This query will return JSON looking like this:

<pre class="prettyprint lang-js">
{
  "took" : 2,
  // ... snip ...
  "hits" : {
    "total" : 4,
    // ... snip ...
  },
  "facets" : {
      "published" : {
        "_type" : "histogram",
        "entries" : [ {
          "time" : 1293840000000,
          "count" : 1
        }, {
          "time" : 1293926400000,
          "count" : 2
        },
        // ... snip ...
        ]
      }
  }
}
</pre>

We are interested in the `facets.published.entries` array, as in the previous example. And again, we will need some HTML to hold our chart and load the data.
Since the mechanics are very similar, please refer to the [full source code](https://gist.github.com/900542/#file_chart.html) for this example.

With the JSON data, it's very easy to create rich, interactive timeline in _Protovis_, by using a customized [_area chart_](http://vis.stanford.edu/protovis/ex/area.html).

The full, annotated code of the `Timeline()` JavaScript function is displayed below.

<pre class="prettyprint lang-js">
// ===========================
// A timeline chart with Protovis
// ===========================
//
// See http://vis.stanford.edu/protovis/ex/area.html
//
var Timeline = function(dom_id) {
  // Set the default DOM element ID to bind
  if ('undefined' == typeof dom_id)  dom_id = 'chart';

  var data = function(json) {
    // Set the data for the chart
    this.data = json;
    return this;
  };

  var draw = function() {

          // Set-up the data
      var entries = this.data;
          // Add the last "blank" entry for proper timeline ending
          entries.push({ count : entries[entries.length-1].count });
          // console.log('Drawing, ', entries);

          // Set-up dimensions and scales for the chart
      var w = 600,
          h = 100,
          max = pv.max(entries, function(d) { return d.count;}),
          x = pv.Scale.linear(0, entries.length-1).range(0, w),
          y = pv.Scale.linear(0, max).range(0, h);

          // Create the basis panel
      var vis = new pv.Panel()
          .width(w)
          .height(h)
          .bottom(20)
          .left(20)
          .right(40)
          .top(40);

           // Add the chart legend at top left
       vis.add(pv.Label)
           .top(-20)
           .text(function() {
             var first = new Date(entries[0].time);
             var last  = new Date(entries[entries.length-2].time);
             return "Articles published between " +
                    [first.getDate(), first.getMonth()+1, first.getFullYear()].join("/") +
                    " and " +
                    [last.getDate(),  last.getMonth()+1,  last.getFullYear()].join("/");
            })
           .textStyle("#B1B1B1")

           // Add the X-ticks
       vis.add(pv.Rule)
           .data(entries)
           .visible(function(d) {return d.time;})
           .left(function() { return x(this.index); })
           .bottom(-15)
           .height(15)
           .strokeStyle("#33A3E1")
           // Add the tick label (DD/MM)
           .anchor("right").add(pv.Label)
            .text(function(d) { var date = new Date(d.time); return [date.getDate(), date.getMonth()+1].join('/'); })
            .textStyle("#2C90C8")
            .textMargin("5")

           // Add the Y-ticks
       vis.add(pv.Rule)
           // Compute tick levels based on the "max" value
           .data(y.ticks(max))
           .bottom(y)
           .strokeStyle("#eee")
           .anchor("left").add(pv.Label)
            .text(y.tickFormat)
            .textStyle("#c0c0c0")

           // Add container panel for the chart
       vis.add(pv.Panel)
           // Add the area segments for each entry
          .add(pv.Area)
            // Set-up auxiliary variable to hold state (mouse over / out) 
           .def("active", -1)
           // Pass the data to Protovis
           .data(entries)
           .bottom(0)
           // Compute x-axis based on scale
           .left(function(d) {return x(this.index);})
           // Compute y-axis based on scale
           .height(function(d) {return y(d.count);})
           // Make the chart curve smooth
           .interpolate('cardinal')
           // Divide the chart into "segments" (needed for interactivity)
           .segmented(true)
           .fillStyle("#79D0F3")

           // On "mouse over", set the relevant area segment as active
           .event("mouseover", function() {
             this.active(this.index);
             return this.root.render();
           })
           // On "mouse out", clear the active state
           .event("mouseout",  function() {
             this.active(-1);
             return this.root.render();
           })
           // On "mouse down", perform action, such as filtering the results...
           .event("mousedown", function(d) {
             var time = entries[this.index].time;
             return (alert("Timestamp: '"+time+"'"));
           })

          // Add thick stroke to the chart
          .anchor("top").add(pv.Line)
           .lineWidth(3)
           .strokeStyle('#33A3E1')

          // Add the circle "label" displaying the count for this day
          .anchor("top").add(pv.Dot)
           // The label is only visible when area segment is active
           .visible( function() { return this.parent.children[0].active() == this.index; } )
           .left(function(d) {return x(this.index);})
           .bottom(function(d) {return y(d.count);})
           .fillStyle("#33A3E1")
           .lineWidth(0)
           .radius(14)
           // Add text to the label
          .anchor("center").add(pv.Label)
           .text(function(d) {return d.count;})
           .textStyle("#E7EFF4")

           // Bind the chart to DOM element
          .root.canvas(dom_id)
           // And render it.
          .render();
  };

  // Create the public API
  return {
    data   : data,
    draw   : draw
  };

};
</pre>

Again, you can see the full example [here](/blog/assets/dashboards/timeline.html). Be sure to check out the documentation on the [_area_](http://vis.stanford.edu/protovis/docs/area.html) primitive in _Protovis_, and watch what happens when you change `interpolate('cardinal')` to `interpolate('step-after')`. You should have no problems to draw a _stacked area chart_ from multiple facets, add more interactivity, and completely customize the visualization.

The important thing to notice here is that the chart fully responds to any queries we pass to _ElasticSearch_, making it possible to simply and instantly visualize metrics such as _“Display publishing frequence of this author on this topic in last three months”_, with a query such as `author:John AND topic:Search AND published:[2011-03-01 TO 2011-05-31]`.


## tl;dr ##

When you need to make rich, interactive data visualization for complex, ad-hoc queries, using data returned by _facets_ from [_ElasticSearch_](http://www.elasticsearch.org/) may well be one of the easiest ways to do it, since you can just pass the JSON response to a toolkit like [_Protovis_](http://vis.stanford.edu/protovis/).

By adapting the approach and code from this article, you should have a working example for your data in couple of hours.
