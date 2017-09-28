# Step by step tutorial

In this section we will take you through a complete setup of a chart, built on the DQF Data Connector.

The goal is to create a [treemap](https://en.wikipedia.org/wiki/Treemapping) that visualizes the volume of a set of target languages, and also indicates the relative productivity in words per hour with the redness of the area. We are interested in these languages: French, German and Spanish, Danish, Dutch and Greek.

First we will pick a tool for creating charts. There are many possibilities, including desktop applications such as Tableau or Microsoft Power BI. There are also libraries, closed or open source, many of them in JavaScript. Plotly, d3.js or ChartJS to name a few.

For this tutorial we'll use Google Charts. It is easy to use and has a very clear documentation (https://developers.google.com/chart/)

This tutorial is not focused on the details of Google Charts. We will explain some of the code here, but for a better understanding of Google Charts, it is better to do a specialized tutorial or course, or to read through the [Google Charts documentation](https://developers.google.com/chart/interactive/docs/).

## Requirements for this tutorial
For this tutorial an api-key is needed. Integrators can ask the [TAUS team](mailto://dqf@taus.net) for this key. Knowledge of JavaScript and html is expected. Also a text editor is needed to write your html file.

## Planning the chart

As  mentioned in the previous paragraph, we want to visualize the volume of a certain set of target languages, and the translation productivity in that particular set of target languages. Productivity is expressed in number of words translated per hour.

Let's visit the Swagger specifications on https://data-connector.stag.taus.net to pick the information we need from the API. In the headings 'Productivity' and 'Segment Count' we find the URLs we need for our chart.

The 'Productivity' request will return the time spent on translation. By adding parameters to the URL, we can filter the response on the target languages we want to focus on, and also group by target language. So the URL we will be using for this part of the data will be http://waiter.dev.ta-us.net/v3/productivity/industry?targetLanguage=fr-FR,de-DE,es-ES,nl-NL,da-DK,nb-NO&groupBy=targetlanguage

The expected response will be something like this:
```
[{
	"group": {
		"targetLanguage": "fr-FR"
	},
	"totalTime": 1234
},{
	"group": {
		"targetLanguage": "de-DE"
	},
	"totalTime": 1234
}, ...
]
```

We need also to know the number of source words, for both the volume of the target language, and to calculate the productivity in number of words per hour.

We can find this also on the Swagger specifications: https://data-connector.stag.taus.net/v3/sourceCount/industry?targetLanguage=fr-FR,de-DE,es-ES,nl-NL,da-DK,nb-NO&groupBy=targetLanguage, and the return is similar to the productivity, but instead of 'totalTime' we get three different key-value pairs for 'segmentCount', 'characterCount', and 'wordCount'.


## Prepare a chart html file
In order to show Google Charts we need to set up a html file that links to the library of Google Charts. Let's follow some of the steps as they are explained in the Google Charts Quick Start (https://developers.google.com/chart/interactive/docs/quick_start).

Use your favorite IDE or text editor to create a new, blank html file. You can start with

```
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>DQF charts tutorial</title>
</head>
<body>

</body>
</html>
```

In the body of the file we'll prepare a location where the chart should go. We do that by placing an empty `div` element with an `id` attribute so we can later refer to this element when we set up the script that creates the chart. For example `<div id="chart_div"></div>`.

After this, we will turn to the `<head>` of the page, where we will need a link to the Google library, and the scripting that will be the main part of this tutorial.

The link is easy:
`<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>`

In a new script element, we will load the library's object and pick up the treemap package here. Next, in a callback function, we'll start the function that we'll be writing in the following steps, as soon as the library is loaded. This script element is like this:
```
<script type="text/javascript">
	// Load the Visualization API and the corechart package.
	google.charts.load('current', {'packages': ['corechart', 'treemap']});

	// Set a callback to run when the Google Visualization API is loaded.
	google.charts.setOnLoadCallback(fetchNewData);
</script>
```

If you followed the tutorial until here, you have this html file:
```
<!DOCTYPE html>
<html lang="en">
<head>
	<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
	<script type="text/javascript">
		// Load the Visualization API and the corechart package.
		google.charts.load('current', {'packages': ['corechart', 'treemap']});

		// Set a callback to run when the Google Visualization API is loaded.
		google.charts.setOnLoadCallback(fetchNewData);
	</script>

	<meta charset="UTF-8">
	<title>DQF charts tutorial</title>
</head>
<body>
<div id="chart_div"></div>
</body>
</html>
```

In order to get the chart showing in the html file, we will finish the functions that deal with creating the chart, and we will feed it with some hard coded data.

## Functions to manipulate the data

For its treemap charts, Google Charts requires data in a certain format. For the exact requirements see: https://developers.google.com/chart/interactive/docs/gallery/treemap

A treemap is a chart with squares of which the size and the color of the square indicates a certain relative value. Each area can consist of sub-data: clicking on such an area will reveal a new patchwork of areas signifying the values of the area's constituting values.  For this tutorial we keep things simple, and apply only one layer of data below the total.

The complete data set for this treemap is an array, in which each element again is an array. The latter array consists of four elements:
1. the name of the row,
2. the 'parent' of the row (as explained the treemap has layers, so groups of arrays can add up to their parent node),
3. a number used for the size of the area,
4. a number used for the color of the area.

So a data set might look like:
```
[
	['language 1', 'global', 1234, 1234],
	['language2', 'global', 1234, 1234],
	['global', null, 1234, 1234]
]
```

For our chart we will want to have the third column to represent the volume of the target language, and the fourth column should represent the productivity of the target language.

In the swagger specifications, we see that the data from the API is returned like this:
```
[{
  "group": {},
  "totalTime": 0
}]
```

and this:

```
[{
  "group": {},
  "segmentCount": 0,
  "characterCount": 0,
  "wordCount": 0
}]
```

So, first we need to find a way to put these data in the form mentioned earlier, to make sure it can be consumed by Google Charts. You can do that in many ways, but for now we'll stick to two functions. One merges the two responses from the requests into a single object, and the other transforms the merged data into the data that is suitable for Google Charts.

First a function that merges source count data, and productivity data in a single merged object:
```
function mergingData (sourceCountData, productivityData) {
    var mergedData = [];
    for (var j = 0; j < sourceCountData.length; j++) {
        for (var i = 0; i < productivityData.length; i++) {
            if (productivityData[i].group.targetLanguage === sourceCountData[j].group.targetLanguage) {
                var length = mergedData.push(productivityData[i]);
                mergedData[length - 1].wordCount = sourceCountData[i].wordCount;
            }
        }
    }
    return mergedData
}
```

With the responses from the Data Connector as parameters for the function, this will return data in a form like this:
```
[{
	"group": {
		"targetLanguage": "fr-FR"
	},
	"totalTime": 1234,
	"wordCount": 1234
}, {
	"group": {
		"targetLanguage": "de-DE"
	},
	"totalTime": 1234,
	"wordCount": 1234
}, ...
]
```

If used as input, the following function will transforms the data to the format needed by the treemap. Notice that we first create a row in which the word count and the time of all the separate language are calculated to produce global numbers. After that the function loops through all the languages to create a row for each language.
```
function prepareTreemapData(mergedData) {
    var transformedData = [];
    var globalWordCount = mergedData.reduce(function (total, currentNumber) {
        return total + currentNumber.wordCount
    }, 0);
    var globalProductivity = mergedData.reduce(function (total, currentNumber) {
        return total + currentNumber.totalTime
    }, 0)/globalWordCount;

    transformedData.push(['global', null, globalWordCount, globalProductivity]);

    mergedData.forEach(function (t) {
        transformedData.push([t.group.targetLanguage, 'global', t.wordCount, t.totalTime])
    });
    return transformedData
}
```

In our html file we now have this:
```
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="utf-8">
	<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
	<script type="text/javascript">
		// Load the Visualization API and the corechart package.
		google.charts.load('current', {'packages': ['corechart', 'treemap']});

		// Set a callback to run when the Google Visualization API is loaded.
		google.charts.setOnLoadCallback(fetchNewData);


        function mergingData (sourceCountData, productivityData) {
            var mergedData = [];
            for (var j = 0; j < sourceCountData.length; j++) {
                for (var i = 0; i < productivityData.length; i++) {
                    if (productivityData[i].group.targetLanguage === sourceCountData[j].group.targetLanguage) {
                        var length = mergedData.push(productivityData[i]);
                        mergedData[length - 1].wordCount = sourceCountData[i].wordCount;
                    }
                }
            }
            return mergedData
        }

        function prepareTreemapData(mergedData) {
            var transformedData = [];
            var globalWordCount = mergedData.reduce(function (total, currentNumber) {
                return total + currentNumber.wordCount
            }, 0);
            var globalProductivity = mergedData.reduce(function (total, currentNumber) {
                return total + currentNumber.totalTime
            }, 0)/globalWordCount;

            transformedData.push(['global', null, globalWordCount, globalProductivity]);

            mergedData.forEach(function (t) {
                transformedData.push([t.group.targetLanguage, 'global', t.wordCount, t.totalTime])
            });
            return transformedData
        }

	</script>
	<meta charset="UTF-8">

	<title>DQF charts tutorial</title>
</head>
<body>
<div id="chart_div"></div>
</body>
</html>
```

## Receiving the data with ajax calls to the Data-Connector
With all the preparation being done, it is now time to actually make the call to the data-connector. This can be done with AJAX calls. An easy way is to use jQuery for that. In order to use jQuery, create a link in the header of the html file like this:
```
	<script src="https://ajax.googleapis.com/ajax/libs/jquery/2.2.2/jquery.min.js"></script>
```

We have to make two separate calls: one call to retrieve the time, and one to retrieve the wordcount. Because we cannot continue until both data points are in, we need a function that waits for both responses before it continues to create the chart. jQuery does this with the method ``$.when(functionA, functionB).then(functionC)``. Let's integrate that in a new 'fetchNewData' function, like this:

```
function fetchNewData() {
    var headers = {'apiKey': '[apiKey goes here]'};
    var productivityRaw;
    var sourceCountRaw;
    var filterGroupingParameters = {groupBy: 'targetLanguage', targetLanguage: 'pl-PL,fi-FI,ba-RU,nl-NL,pt-BR,it-IT,en-US,ru-RU,zh-CN,ja-JP,es-MX,es-ES,pt-PT'}
    $.when(
        $.ajax({
            type: "GET",
            headers: headers,
            data: filterGroupingParameters,
            url: "http://waiter.dev.ta-us.net/v3/productivity/industry",
            async: false,
            error: function(){
                console.log('Data be found');
            },
            success: function (data) {
                productivityRaw = data;
            }
        }),
        $.ajax({
            type: "GET",
            headers: headers,
            data: filterGroupingParameters,
            url: "http://waiter.dev.ta-us.net/v3/sourceCount/industry",
            async: false,
            error: function(){
                console.log('Data be found');
            },
            success: function (data) {
                sourceCountRaw = data;
            }

        })
    ).then(function () {
        var mergedData = mergingData(sourceCountRaw, productivityRaw);
        var treemapData = prepareTreemapData(mergedData);
        createChart(treemapData);
    })
}
```

This function uses both URLs, the parameters and the API key to retrieve the data from the Data Connector. Once the data are retrieved, both the mergingData function and the prepareTreemapData function are called. With the transformed data the createChart function can be called.

The createChart function is the final bit. It uses the data to create a datatable for the google visualization library, and then draws the visualization with this datatable in the 'chart_div' element on the page.

```
function createChart(chartData) {
	var data = new google.visualization.DataTable();
	data.addColumn('string', 'Target Language');
	data.addColumn('string', 'Parent');
	data.addColumn('number', 'Volume');
	data.addColumn('number', 'Productivity');
	data.addRows(chartData);
	var chart = new google.visualization.TreeMap(document.getElementById('chart_div'));

	chart.draw(data, {
		minColor: '#888888',
		midColor: '#aa4444',
		maxColor: '#cc0000',
		generateTooltip: showFullTooltip
	});

	function showFullTooltip(row) {
		return '<div style="background:#fd9; padding:10px; border-style:solid">' +
			'<span><b>' + data.getValue(row, 0) +
			'</b>,</span><br>' +
			data.getColumnLabel(2) + ': ' + data.getValue(row, 2) + '<br>' +
			data.getColumnLabel(3) + ': ' + data.getValue(row, 3) + ' </div>';
	}

}
```

If you followed all the steps in this tutorial, you have this file:
```
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="utf-8">
	<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
	<script src="https://ajax.googleapis.com/ajax/libs/jquery/2.2.2/jquery.min.js"></script>
	<script type="text/javascript">
		// Load the Visualization API and the corechart package.
		google.charts.load('current', {'packages': ['corechart', 'treemap']});

		// Set a callback to run when the Google Visualization API is loaded.
		google.charts.setOnLoadCallback(fetchNewData);

        function fetchNewData() {
            var headers = {'apiKey': '[apiKey goes here]'};
            var productivityRaw;
            var sourceCountRaw;
            var filterGroupingParameters = {groupBy: 'targetLanguage', targetLanguage: 'pl-PL,fi-FI,ba-RU,nl-NL,pt-BR,it-IT,en-US,ru-RU,zh-CN,ja-JP,es-MX,es-ES,pt-PT'}
            $.when(
                $.ajax({
                    type: "GET",
                    headers: headers,
                    data: filterGroupingParameters,
                    url: "http://waiter.dev.ta-us.net/v3/productivity/industry",
                    async: false,
                    error: function(){
                        console.log('Data be found');
                    },
                    success: function (data) {
                        productivityRaw = data;
                    }
                }),
                $.ajax({
                    type: "GET",
                    headers: headers,
                    data: filterGroupingParameters,
                    url: "http://waiter.dev.ta-us.net/v3/sourceCount/industry",
                    async: false,
                    error: function(){
                        console.log('Data be found');
                    },
                    success: function (data) {
                        sourceCountRaw = data;
                    }

                })
            ).then(function () {
                var mergedData = mergingData(sourceCountRaw, productivityRaw);
                var treemapData = prepareTreemapData(mergedData);
                createChart(treemapData);
            })
        }


        function mergingData (sourceCountData, productivityData) {
            var mergedData = [];
            for (var j = 0; j < sourceCountData.length; j++) {
                for (var i = 0; i < productivityData.length; i++) {
                    if (productivityData[i].group.targetLanguage === sourceCountData[j].group.targetLanguage) {
                        var length = mergedData.push(productivityData[i]);
                        mergedData[length - 1].wordCount = sourceCountData[i].wordCount;
                    }
                }
            }
            return mergedData
        }


        function prepareTreemapData(mergedData) {
            var transformedData = [];
            var globalWordCount = mergedData.reduce(function (total, currentNumber) {
                return total + currentNumber.wordCount
            }, 0);
            var globalProductivity = mergedData.reduce(function (total, currentNumber) {
                return total + currentNumber.totalTime
            }, 0)/globalWordCount;

            transformedData.push(['global', null, globalWordCount, globalProductivity]);

            mergedData.forEach(function (t) {
                transformedData.push([t.group.targetLanguage, 'global', t.wordCount, t.totalTime])
            });
            return transformedData
        }


		function createChart(chartData) {
			var data = new google.visualization.DataTable();
			data.addColumn('string', 'Target Language');
			data.addColumn('string', 'Parent');
			data.addColumn('number', 'Volume');
			data.addColumn('number', 'Productivity');
			data.addRows(chartData);
			var chart = new google.visualization.TreeMap(document.getElementById('chart_div'));

			chart.draw(data, {
				minColor: '#888888',
				midColor: '#aa4444',
				maxColor: '#cc0000',
				generateTooltip: showFullTooltip
			});

			function showFullTooltip(row) {
				return '<div style="background:#fd9; padding:10px; border-style:solid">' +
					'<span><b>' + data.getValue(row, 0) +
					'</b>,</span><br>' +
					data.getColumnLabel(2) + ': ' + data.getValue(row, 2) + '<br>' +
					data.getColumnLabel(3) + ': ' + data.getValue(row, 3) + ' </div>';
			}
		}
	</script>
	<meta charset="UTF-8">
	<title>DQF charts tutorial</title>
</head>
<body>
<div id="chart_div"></div>
</body>
</html>
```

Depending on the data that the data-connector returns, you might get a chart like this
![enter image description here](https://lh3.googleusercontent.com/-Das3U_9Haeg/WaaKGwGt5oI/AAAAAAAAAds/zCXm3HLD-w0QuRVghtIxhadl8_kn1IjwgCLcBGAs/s0/treemapSample.png "treemapSample.png")