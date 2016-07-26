## Disclaimer ##
This are my personal work notes on the tutorial. They will most likely end up being a copy of Mike Bostock's original content. This is how I work, re writting stuff let's me write properly inside my head. They will also most probably contain side notes and references that I found useful along the way.

#Mike Bostock's "Let's Make a Bar Chart", I#
Let say we have the following array of numbers:

	var data = [4, 8, 15, 16, 23, 42];

A simple way to visualize such data is a bar chart. The tutorial covers how to create such a chart using [D3 JavaScript library](http://d3js.org/). We strat by creating a bare-bones version in HTML, then move to a more complete chart in SVG and alstly animate the transitions between the views.

## Selecting an Element ##
In vanilla JS elements are dealt with usually one at a time. For example, if I were to create a `div` element, set its contents and append it to the `body` we could do something like this:

	var div = document.createElement("div");
	div.innerHTML="Hellom world!";
	document.body.appendChild(div);

With D3 we handle instead groups of related elemetns called **selections**. Selections allors us to work with element *en masse*, it is possible to manipulate a single or many elements at the same time without substantially restructuring the code.

A selection can be created with alot of different methods. The most common one is probaly querying a selector. A [selector](http://www.w3.org/TR/selectors-api/) is a  special string that idientifies desired elements by property, say by name or class (`"div"` or `.foo`, respectively). It is possible to create a slection for a single element:

	var body = d3.select("body");
	var div = body.append("div");
	div.html("Hello world!");

It is of course also possible to perform the same kind of operation pn many elements:

	var section = d3.selectAll("section");
	var div = section.append("div");
	div.html("Hello, world!");

## Chaining Methods##
Another convenience of selectio s is *method chaining*: Selction methods (such as `selction.attr`) return the current selection. This allows to apply multiple operations to the same elements.

To set the text color and bakcground of thebody without chaining one would have to say:

	var body = d3.select("body")
	body.style("color", "white")
	body.style("background-color", "white")

With method chaining this becomes:

	d3.select("body")
		.attr("color", "white")
		.attr("background-color", "black");

There is a gotcha however with method chaining. While most operations return the same selection, there are some that return a new selection. For example `selection.append` returns a new selection containing the new elements. Ex:

	d3.select("body")
			.style("color", "white")
			.style("background-color", "black")
	.append("div")
			.html("Hello blue baby from the div")
			.style("color", "yellow")
			.style("background-color", "blue")

Method chaining can only be used to descend into the docuent hierarchy, use var to keep references to selections and go back up:

	var section = d3.selectAll("section")

	section.append("div")
		.html("First!")

	section.append("div")
		.html("Second.")

## Coding a Chart, manually ##
Let's create a chart *without* JavaScript. After all, there are only six numbers in the dataset. So it's not hard to this by hand with a few `div` elements, set their width as a multiple of the data and be done with that:

	<style>
	.chart div {
		font: 10px sans-serif;
		background-color: steelblue;
		text-align: right;
		padding: 3px;
		margin: 1px;
		color: white;
	}
	</style>

	<div class="chart">
		<div style="width: 40px;">4</div>
		<div style="width: 80px;">8</div>
		<div style="width: 150px;">15</div>
		<div style="width: 160px;">16</div>
		<div style="width: 230px;">23</div>
		<div style="width: 420px;">42</div>
	</div>

This chart has a `div` for a container and one child `div` for each bar. The implementation can be simplified further by removing the `div` container. But more commonly the page will contain additional information. Thus having a chart container will allow to position and style the chart without affecting the rest of the page.

##Coding a chart, automatically##
Of course hard-coding is impractical for most dataset, and the objective is to create charts from data automatically. Let us first create the identical structure using D3.

We start from an empty page containing only a `div` of class `"class"`. The following script selects the chart container and then append a child div for each bar with the desired width:

	<div class="chart"></div>

	<script>
		var data = [4, 80, 15, 16, 23, 42];
			
		d3.select(".chart")
			.selectAll("div")
			.data(data)
		.enter().append("div")
			.style("width", function(d) {return d * 10 + "px";})
			.text(function (d) {return d;})

	</script>

This code introduces an important new concept. The **data join**. Let's break it down by writing the previous concise code in long form to see how it works.

* First, we select the chart container using a class selector.

	var chart = d3.select(".chart");

* Now we initiate the data join by defining the selection to which will join the data

	var bar = chart.selectAll("div")

The data join is a general pattern that can be used to create, update or destroy elements whenever data changes. This approach might feel odd but the benefit is that we only need to learnd and use a single apttern to manipulate the page. Thus, whether we are building a static chart or a dynamic one the code remains roughly the saim

Think of the initial selection as declarring the elemens we want to exist ([**Thinking with joins**](https://bost.ocks.org/mike/join/)).

* Now we join the data to the selection using `selection.data`:

	var barUpdate = bar.data(data);

Since the selection is empty, the returned *update* and *ext* selections are also empty and we need only handle the *enter* selection which represents new data for which  there was no existing element (?? [Enter, Update, Exit _Christian Behrens](https://medium.com/@c_behrens/enter-update-exit-6cafc6014c36#.l4cwzsxuc), [General Update Pattern, 1 _ Mike Bostock](https://bl.ocks.org/mbostock/3808218), [Learn D3 Part2: Enter and Exit _ Scott becker](http://synthesis.sbecker.net/articles/2012/07/09/learning-d3-part-2) ??).

The missing elements are instantiated by appending to the enter selection:

	var barEnter = barUpdate.enter().append("div");

* Now we set the width of each new bar as a multiple of the associated data value, d.

	barEnter.style("width", function (d) {return d * 10 + "px";});

Because these elements were created with the data join, each bar is already bound to data.  We set the dimensions of each bar based on its data.

* Lastly we use a function to set the text to be displayed on each bar and produce a label:

	barEnter.text(function (d) {return d;});

D3's selection operators such as `attr`, `style` and `property` allow to specify the value either as a constant (the same value for everyone) or a function (computed separeley for each element). 

## Scaling to fit##
One weakness of the code above is the *magic number* 10, which is used to scale the data value to the appropriate pixel width. This number depends on the domain of the data (minimum and maximum values, 0 and 42 in the example) and the desired width of the cart (420)

We can make these dependencies explicit and eliminate the magic number bu using a *linear scale*. D3's scales specify a mapping from data space (**domain**) and display space (**range**).

	var x = d3.scale.linear()
		.domain([0, d3.max(data)])
		.range([0, 420])

Although x looks like an object, it is also a function that returnsthe scaled display value in the range for a givend ata value in the domain. An input of 4 will return 40, 16 will return 160.

To use this new and fancy scale we simply we replace the hard coded part in te above code:

	d3.select(".chart")
		.selectAll("div")
			.data(data)
		.enter().append("div")
			.style("width", function (d) {return x(d) + "px"; });
			.text(funciton(d) {return d; });


#Mike Bostock's "Let's Make a Bar Chart", II#
The first part of the tutorial covered making a basic chart using HTML. in this part we extend the example using SVG and make it more realistc by loading an actual external data file in TSV format.

##Introducing SVG##
HTML is limited to rectangular shapes. SVG on the other hand supports powerful drawing primitives.

We won't need SVG's extensive features for a lowly bar chart, but learning it is worthwile. 

##Coding a Chart manually##
Before we construct the new chart using JS, let's revisit the static specification using SVG.

	<style>
	.chart rect{
		fill: steelblue;
	}

	.chart text {
		fill: white;
		font: 10px sans-serif;
		text-anchor: end;
	}
	</style>

	<!-- <script src="https://d3js.org/d3.v4.min.js"></script> -->

	<body>
		<p>Hello World!</p>
		
		<svg class="chart" width="420" height="120">
			<g transform="translate(0, 0)">
				<rect width="40" height="19"></rect>
				<text x="37" y="9.5" dy=".35em">4</text>
			</g>

			<g transform="translate(0, 20)">
				<rect width="80" height="19"></rect>
				<text x="77" y="9.5" dy=".35em">8</text>
			</g>

			<g transform="translate(0, 40)">
				<rect width="160" height="19"></rect>
				<text x="157" y="9.5" dy=".35em">16</text>
			</g>

			<g transform="translate(0, 60)">
				<rect width="150" height="19"></rect>
				<text x="147" y="9.5" dy=".35em">15</text>
			</g>

			<g transform="translate(0, 80)">
				<rect width="230" height="19"></rect>
				<text x="227" y="9.5" dy=".35em">23</text>
			</g>

			<g transform="translate(0, 100)">
				<rect width="420" height="19"></rect>
				<text x="417" y="9.5" dy=".35em">42</text>
			</g>
		</svg>
		
	</body>

As before a stylesheet applies colors and others aesthetic properties to the svg elements. but unlike the `div` elements that were implicitly positioned using flow layout, the SVG elements must be absolutely positioned with hard code translations relative to the origin.

A common point of confusion in SVG is distinguishing between properties that must be specified as attributes and properties that can be be set as styles. There is a full list of [styling properties](http://www.w3.org/TR/SVG/styling.html). But a quick rule of thumb is that geometry (`width`, `height`) must be specifed as attributes while aesthetics (such as `fill` color) can be specifed as styles. While attributes can be used for anything it is recommended to prefer styles for aesthetics; this ensures any inline styles play niceley with cascading stylesheets.

SVG requires text to be placed explicitly in `text` elements. Since `text` elements do not support padding or margins, the text position must be offset by three pixels from the end of the bar, while the `dy`  offset is used to center the text vertically.

##Coding a chart, automatically##
Let us construct the same chart automatically:

	<svg class="chart"></svg>

	<script src="https://d3js.org/d3.v4.min.js"></script>		
	<script>
		var data = [4, 8, 15, 16, 23, 42];

		var chartWidth = 420,
			barHeight = 20;

		var x = d3.scaleLinear()
			.domain([0, d3.max(data)])
			.range([0, chartWidth]);

		var chart = d3.select(".chart")
			.attr("width", chartWidth)
			.attr("height", barHeight * data.length)

		var bar = chart.selectAll("g")
			.data(data)
			.enter().append("g")
				.attr("transform", function (d, i) { return "translate(0," + i * barHeight + ")"})
		
		bar.append("rect")
			.attr("width", x)
			.attr("height", barHeight - 1)

		bar.append("text")
			.attr("x", function(d) { return x(d) - 3;})
			.attr("y", barHeight / 2)
			.attr("dy", ".35em")
			.text(function (d) {return d;});
	</script>

We set the svg element's size in JS so that we can compute the height based on the size of the data set (`data.length`). This way the size is absed on the height of each bar rather than the overall height of the chart.

Each bar consists of a `g` element which in turn contains `rect` and `text` elements. We use a data join (an enter selection) to create a `g` element for each data point. We then translate the `g` element vertically, creating a local origin position the bar and its associated label.

Since there is exactly one `rect` and one `text` per `g` element, we can append these elements directly to the `g` without needing additional joins. Data joins are only needed when creating a variable number of children based on data. Here we are appending just one child per parent. The appended `rects` and `texts` inherit data from their parent `g` element, and thus we can use data to compute the bar width and label position.

##Loadind data##
Let's make this data more realistic by extracting the dataset into a separate file. An external data file separates the chart implementation from its data, making it easy to reuse the implementation on multiple datasets or even live data that changes over time.

TSV is a convenient tabular data format. Each line represents a table row, where each row consists of multiple columns separated by tabs. the first line is the header row and specifies the column names. Whereas our previous data set was a simple array of numbers, now we'll add a descriptive name column. Our data file now looks like this:

	name 	value
	Locke	4
	Reyes	8
	Ford	15
	Jarrah	16
	Shepard	23
	Kwon	42

To use this data in a web browser we need to download the file from a web server and then parse it,converting the text file into usable JS objects.Fortunately this taks can be taken care of ny a single function `d3.tsv`.

loading data introduces a new complexity. Downloads are *asynchronous*. When we call `d3.tsv` it returns immediately while the file downloads in the background. At some point in the future, when the download finishes, the callback function is invoked with the new data, or an error if the download failed. In effect the code is evaluated out of order:

	// 1. Code here runs first, before the download starts.

	d3.tsv("data.tsv:, function(eror, data){
		// 3. Code here runs last, after the download has finished
	});

	// 2. Code here runs second, while the file is downloading.

Thus we need to separate the chart implementation into two phases. 

* First, we initialize as much as we can when the page loads, so that the page does not reflow after the data downloads.

* Second, we complete the remainder of the chart inside the callback function.

Let's restructure the code:

	<script>
		var chartWidth = 420,
			barHeight = 20;

		var x =d3.scaleLinear()
			.range([0, chartWidth]);

		var chart = d3.select(".chart")
			.attr("width", chartWidth);

		d3.tsv("data.tsv", type, function(error, data) {
			x.domain([0, d3.max(data, function(d) { return d.value; })]);
			
			chart.attr("height", barHeight * data.length);

			var bar = chart.selectAll("g")
				.data(data)
			  .enter().append("g")
				.attr("transform", function (d, i) { return "translate(0," + i * barHeight + ")"; });

			bar.append("rect")
				.attr("width", function (d) { return x(d.value); })
				.attr("height", barHeight - 1)

			bar.append("text")
				.attr("x", function(d) { return x(d.value) - 3; })
				.attr("y", barHeight / 2)
				.attr("dy", ".35em")
				.text(function (d) {return d.value; });
		});

		function type(d) {
			d.value = +d.value; //coerce to number
			return d;
		}

	</script>

What changed this time? 

* The `x`scale is declared in the same palce as before. but a slong as the data hasn't been loaded it is impossible to define its domain. We neeed the data to set it and thus it is done inside the callback function.

* Likewise the height of the cahrt depends on the data. it is also set insdie the callback.

* Now our dataset contains both the names and values. We must refer to the value as `d.value` rather than jsut `d` as before. Each data point is an object rather than a number.

* Any place in the old implementation where we referred to `d` must now refer to `d.value`. in particular wheras before we could pass `x` as a function to compute the width of the bar, now we must specify a function that passes the data value to the scale: `function (d) {return x(d.value); }`.

* Likewise when computing the maximum value from the dataset, we must pass an [accessor](http://stackoverflow.com/questions/26330927/what-is-accessor-function) function to `d3.max` telling it to evaluate each data point.

* There is yet another gotcha with external data that we need to discuss: **data types**!! The name column containes strings while the column value contains numbers. `d3.tsv` is not smart enough` to detect and handle different types automaticaly. Instead we specify a `type` function that is passed as second argumnt to `d3.tsv`.  This type function can modify the data object representing each row, modifying or converting it to a more suitable represention.

The type conversion isn't strictly required. But it's an awfully good idea. by default all columns in TSV and CSV files are strings. If one forgets to convert strings to numbers JavaScript might behave in "weird"/**unexpected** ways (say return `"12"` for `"1" + "2"` rather than `3`).



# Mike's Bostock "Lets's Make a Bar Chart", III #
in the prievious sections we saw how to make a basuc chart in HTML and in SVG. this time we' ll improve the display by rotating the chart into columns and add some axis. We'll also switch to a real dataset, shiwing the relative frequencies of letters in the english language.

##Rotating into columns##
Rotating the columns involves mainly to swapping `x` with `y`. howeever a number of smaller incidental modifications have also to be taken into account. This is the cost of working with SVG rather than with a high level visualization grammer likke ggplot2. On the other hand SVG offers great customizability. And SVG is a web stands.

* When renaming the `x` scale to the `y` scale, the range becomes `[height, 0]` instead of `[0, width]`. This is because th eorigin og SVG's coordinates is in the top-left corner. We want the zero-value to be positioned at the bottom of the cart, rather than the top.

* Likewise, we need to position the bar `rects` by setting the `"y"` and `"height"` attributes, whereas before we only needed to set the `"width"` attribute.

* Previously we multiplied the `var barHeight` by the idnex of the data point (0, 1, 2, ...) to produce fixed-height bars. The resulting chart's this depended on the size of the dataset. Here we want the opposite behavior. The cart size is fixed and we want the bars to adapt to this constraint.

* Lastly the bar labels must be positioned differently for columns rather than bars, cetered jsut below the top f the column. The new `"dy"` attribute value of `".75em"`  anchors the label at approximately the text's [cap height](https://en.wikipedia.org/wiki/Cap_height) rather than the bsaeline.

	<script>
		var chartWidth = 480,
			chartHeight = 400;

		var y =d3.scaleLinear()
			.range([chartHeight, 0]);

		var chart = d3.select(".chart")
			.attr("width", chartWidth)
			.attr("height", chartHeight);

		d3.tsv("data.tsv", type, function(error, data) {
			y.domain([0, d3.max(data, function(d) { return d.value; })]);
			
			var barWidth = chartWidth / data.length;

			var bar = chart.selectAll("g")
				.data(data)
			  .enter().append("g")
				.attr("transform", function (d, i) {return "translate(" + i * barWidth + ", 0)"; });

			bar.append("rect")
				.attr("y", function (d) { return y(d.value); })
				.attr("height", function (d) { return chartHeight - y(d.value); })
				.attr("width", barWidth - 1);

			bar.append("text")
				.attr("x", barWidth / 2 )
				.attr("y", function (d) { return y(d.value) + 3; })
				.attr("dy", ".75em")
				.text(function (d) {return d.value; });
		});

		function type(d) {
			d.value = +d.value; //coerce to number
			return d;
		}

	</script>

# Encoding Ordinal Data #
Unlike *quantitative* values that can be compared, substracted, divided, ... numerically, *ordinal* values are compared by rank. Letters are ordinal, A occurs before B, B before C ,... Wheres as D3's linear, pow and log scales serve to encode quantitative data the `ordinal scales` encode ordinal data. We can thus use an ordinal scale to simplify the positioning of bars by letter.

In its most explicit form, an ordinal scale is a mapping from a discrete set of values (such as names) to a corresponding discrete set of display values (such as pixel positions). Like quantitative scales these sets are called the *domain* and *range* respectively:

	var x = d3.scale.ordinal()
		.domain(["A", "B", "C", "D", "E", "F"])
		.range([0, 1, 2, 3, 4, 5])

The result of `x("A")` is `0`, `x("B")` is `1` and so on. In specifying the domain and range, all that matters is the order of values. Element `i` in the domain is mapped to element `i` in the range.

It would be tedious to enumerate the positions of each bar by hand, we can convert a continuous range into a discrete set of values using `rangeBand` or `rangePoints`.
The rangeBand method computes range values so as to divide the cart area into evely-spaced, envely-sized bands,as in a bar chart. The similar method rangePoints computes envely-spaced range values as in a scatterplot. 

For example

	var x = d3.scale.ordinal()
		.domain(["A", "B", "C", "D", "E", "F"])
		.rangeBands([o, width]);

If `width` is `960` then `x("A")` is now `0` and `x("B")` is `160`. These positions serve as the left edge of each bar, while `x.rangeBand()`  returns the width of each bar. `rangeBands` can also add padding between bars with an optional third argument. The `rangeRoundBands` variant will compute positions that snap to exact pixel boundaries for crisp edges. Compare:

	var x = d3.scale.ordinal()
		.domain(["A", "B", "C", "D", "E", "F"])
		.rangeRoundBands([0, width], .1);

Now `x("A")` is `17` and each bar is 141 pixels wide. And rather thanr hard coding the letters in the domain we can compute them fron the data using `array.map`  and `array.sort`.

	<script>
		var chartWidth = 960,
			chartHeight = 500;

		var x =d3.scaleBand()
			.rangeRound([0, chartWidth])
			.paddingInner(0.01);

		var y =d3.scaleLinear()
			.range([chartHeight, 0]);

		var chart = d3.select(".chart")
			.attr("width", chartWidth)
			.attr("height", chartHeight);

		d3.csv("data.csv", type, function(error, data) {
			x.domain(data.map(function (d) { return d.name; }));
			y.domain([0, d3.max(data, function(d) { return d.value; })]);
			
			var bar = chart.selectAll("g")
				.data(data)
			  .enter().append("g")
				.attr("transform", function (d, i) {return "translate(" + x(d.name) + ", 0)"; });

			bar.append("rect")
				.attr("y", function (d) { return y(d.value); })
				.attr("height", function (d) { return chartHeight - y(d.value); })
				.attr("width", x.bandwidth());

			bar.append("text")
				.attr("x", x.bandwidth() / 2)
				.attr("y", function (d) { return y(d.value) + 3; })
				.attr("dy", ".75em")
				.text(function (d) {return d.value; });
		});

		function type(d) {
			d.value = +d.value; //coerce to number
			return d;
		}
	</script>

## Preparing margins ##
Ordinal scales are often used in conjunction with D#'s axis component to quickly display labeled tick marks, improving the chart's legibility. But before adding an axis we need to clear some space in the amrgins.

By **convention** margins in D3 are specified as an object with top, right, bottom and left properties. The *outer* size of the chart area, which includes the margins, is used to compute the inner size available for graphical marks by substraction the  margins. For example, reasonable values for a 960X500 chart are:

	var margin = {top: 20, right: 30, bottom: 30, left: 40},
		chartWidth = 960 - margin.left - margin.right,
		chartHeight = 500 - margin.top - margin.bottom;

Thus 960 and 500 are the outer width and height, respectively, while the computed inner width and height are 890 and 450. These inner dimensions are then used to initialize scale ranges.

To apply the margins to the sVG container we set the width and height of the SVG element to the outer dimensions. Then we add a `g` element to offset the origin of the cart area by the top-left margin:

	var chart = d3.select(".chart")
		.attr("width", chartWidth + margin.left + margin.right)
		.attr("height", chartHeight + margin.top +margin.bottom)
	  .append("g")
	  	.attr("transform", "translate(" + margin.left + "," + margin.top + ")")
	
Any elements subsequently added to `chart` thus inherit the margins.

##Adding axes##
We define an axis by binding it to our existing *x*-scale and declaring one of the four possible orientations. Seince we want our x-axis to appear below the bars we use the `"bottom"` orientation.

	var xAxis = d3.svg.axis()
		.scale(x)
		.orient("bottom");

The resulting `xAxis` object can be used to render mutiple axes by repeated application using `selection.call`. 

The axis elements are positioned relative to a local origin, so to transform into the desired position we set the `"transform"` attribute on a containing `g` element:

	chart.append("g")
		.attr("class", "x axis")
		.attr("transform", "translate(0," + height + "")")
		.call(xAxis);

The axis container should also have a class name so that we can apply styles. The name `"axis"` here is arbitraty. Multiple class names, such as `"x axis"` are useful for stylin axes differently by dimension while retaining some shared style across dimensions.

The axis component consists of a `path` element which displas the domain, and multiple `".tick"` elements for each tick mark. A tick in turn contains a `text` label and a `line` mark. Most of D#'s examples therefor use the following minimalist style:

	.axis text {
		font: 10px sans-serif;
	}

	.axis path

	.axis line{
		fill:none;
		stroke: #000;
		shape-rendering: crispEdges;
	}

Beyond styles, the appearance of an axis can be further modified by selecting its elements and modifying them after the axis has been created. 

We then get the following code with axis:

	<style>
		.chart rect{
			fill: steelblue;
		}

		.axis text {
			font: 10px sans-serif;
		}

		.axis path,
		.axis line {
			fill: none;
			stroke: #000;
			shape-rendering: crispEdges;
		}

		.x.axis path {
			display: none;
		}
	</style>

	<script>
		var margin = {top: 20, right: 30, bottom: 30, left: 40},
			width = 960 - margin.left - margin.right,
			height = 500 - margin.top - margin.bottom;

		var x =d3.scaleBand()
			.rangeRound([0, width])
			.paddingInner(0.05);

		var y =d3.scaleLinear()
			.range([height, 0]);


		var xAxis = d3.axisBottom()
			.scale(x);

		var yAxis = d3.axisLeft()
			.scale(y);
		


		var chart = d3.select(".chart")
			.attr("width", width + margin.left + margin.right)
			.attr("height", height + margin.top +margin.bottom)
		  .append("g")
		  	.attr("transform", "translate(" + margin.left + "," + margin.top + ")")

		d3.tsv("data.tsv", type, function(error, data) {
			x.domain(data.map(function (d) { return d.letter; }));
			y.domain([0, d3.max(data, function(d) { return d.frequency; })]);

			//Add xAxis
			chart.append("g")
				.attr("class", "x axis")
				.attr("transform", "translate(0," + height + ")")
				.call(xAxis);

			//Add yAxis
			chart.append("g")
				.attr("class", "y axis")
				.call(yAxis);

			// And add the bars			
			var bar = chart.selectAll(".bar")
				.data(data)
			  .enter().append("rect")
			  	.attr("x", function (d) { return x(d.letter) })
				.attr("y", function (d) { return y(d.frequency); })
				.attr("height", function (d) { return height - y(d.frequency); })
				.attr("width", x.bandwidth())
		});

		function type(d) {
			d.frequency = +d.frequency; //coerce to number
			return d;
		}
	</script>




# Mike's Bostock "How Selections Work"#
I now follow Mike Bostock's [article on selections](https://bost.ocks.org/mike/selection/).

The article's follows a comprehensive approach; rather than saying *how to use* selections, the idea is to try and explain how selection are *implemented*.

The articles describes the internal workings of selections rather than the design motivations. Why are such mechanisms needed? Because it is easier to lay down all the pieces first and then explaining how everything works in tandem.

D3 is a visulization library. The article incorporates visual explanations to accompany the text. n subsequent diagrams, the left side shows the structure of selections, while the right sde shows the structure of the data:

## A subclass of Array##
Selections are not arrays of DOM elements. 

* Selections are a *subclass* of array: This sublcass provides methods to manipulate selected eements, such as setting `attributes` and `styles`. Slections inherit native array ethodsas well, such as `array.forEach` and `array.map`. However native methods are not usually used since d3 provides convenient alternatives, such as `selection./each`. (Some native emthods are overridden to adapt their behavior to selections, namely `selection.filter` and `selection.sort`).

## Grouping Elements ##
Another reason selections are not literraly arrays is that they are *arrays of arrays* of elements: a selection is an array of groups, each group is an array of elements. For example `d3.select` returns a selection with one group containing the selected element.

	var selection = d3.select("body");

The group can be inspected as `selection[0]` and the node as `selection[0][0]`. While accessing a node directly is supported, it is more common to use `selection.node`.

Likewise `d3.selectAll` returns a selection with one group and any number of elements:

	d3.selectAll("h2");

Selections returned by `d3.select` and `d3.selectAll` have exactly one group. The only way to obtain a selection ith multiple groupes is `selection.selectAll`. For example if we chose all table rws and then select the rows' cell, then we'll get a group of sibling cells for each row:

	d3.selectAll("tr").selectAll("td")

with selectAll **every element in the old selection becomes a group in the new selection**. Each group contains an old element's matching descendant elements. 

Each group has a `parentNode` property which stores the shared parent of all the group's elements. The parent node is set when the group is created. Thus if one calls `d3.selectAll("tr").selectAll("td")` the returned selection contains groups of `td` elements, whose aprents are `tr` elements. For selections returned by `d3.select` and `d3.selectAll`, the parent element is the document element.

Most of the time it is safe to ignore that selections are grouped. When a function like `selection.attr` or `selection.style` is called, the function is called for each element. The main difference with grouping is that the second argument to the function `(i)` is the within-group index rather than the within-selection index.

##Non-grouping operations##
Only `selectAll` has special behavior regarding grouping. `select` preserves the existing grouping. The select method differs because there is exactly one element in the new selection for each element in the old selection. Thus, `select` also propagates data from parent to child, whereas `selectAll` does not (thus the need for a data-join.)

the `append` and `insert` methods are wrappers on top of `select`, they also preserve grouping and propagate data. Given a document with four sections

	d3.selectAll("section")

If we append a paragraph element to each section, the new selection likewise has a single group with four elements:

	d3.selectAll("section").append("p")

Note that the `parentNode` for this selection is still the document element because `selection.selectAll` has not been called to regroup the selection.

## Null Elements ##
Groups can contain nulls to indicate missing elements. nulls are ignored for most operations, ex: D3 skips null elements when aplying styles and attributes.

null elemetns can occur when `selection.select` cannot find a matching elemtn for a given selector. The `select` method ust preserve the grouping structure sot it fill the missing slots with null. For example if only the last two sections (in the rpevious example) have asides:

	d3.selectAll("section").select("aside");

We will see two null in the selection (preservation of grouping!!)

## Bound to data ##
Surprinsingly data **is not** a property of the selection, but a property of its elements. This means that when data is bound to a selection, the data is stored in the DOM rather than in the selection: data is assigned to the `__data__` property of each element. if an element lack this property, the associated datum is undefined. Data is therefore persistent while selections can be cosnidered transient: we can reselect elements from the DOM and they will retain whatever data was previously bound to them.

Data is bound to elements one of several ways:

* Joined to groups of elements via `selection.data`
* Assigned to individual elemtns via `selection.datum`
* Inherited from a parent via `append`, `insert` or `select`.

While there is no reason to set the `__data__` property directly when it's possible to use `selection.datum` doing so illustrates how data binding is implementded:

	document.body.__data__ = 42;

The D3-idiomatic wy of doing this binding is:

	d3.select("body").datum(42);

If we now append an element to the body, the child automatically inherits data from the parent:

	d3.select("body").datum(42).append("h1");

Andthis brings us to the last method of binding data: the misterious join! But before achieving enlightment we must answer one existential question:

## What is Data??##

Data in D3 can be any array of values. For example an array of numbers:

	var numbers = [4, 5, 2, 9];

An array of objects:

	var letters = [
	  {name: "A", frequency: .08167},
	  {name: "B", frequency: .01492},
	  {name: "C", frequency: .02780},
	  {name: "D", frequency: .04253},
	  {name: "E", frequency: .12702}
	];

Even an array of arrays:

	var matrix = [
	  [ 0,  1,  2,  3],
	  [ 4,  5,  6,  7],
	  [ 8,  9, 10, 11],
	  [12, 13, 14, 15]
	];

The visual representation of selections can be mirrored to represent data.

Just as `seection.style` takes either a constant string to define a uniform style property` (*e.g* `"red"`) for every selected element, or a function for dynamic style per-element, `selection.data` can accept either a constant value or a function.

However, unlike the other selection methods, `selection.data` **defines data per-group rather than per-element**. Data is expressed as an array of values for the group, or a function that returns such an array. Thus, a grouped slection has correspondingly grouped data!!!

## The ey to enlightment ##
To join data to elements, we must know which datum should be assigned to which element. This is done by pairing keys. A key is simply an identifying string, such as  a name; when the key for the datum and an element are equal, the datum is assigned to that element.

The simplest method of assigning keys is by index: the first datum and the first element have the key "0", the seonc d datum and lemtn have the key "1" and so on and so forth.

Joining by index is convenient of the data and elements are in the same order. However, when orders differ, joining by index is insufficient! in this case it is possible to specify a [key function](https://bost.ocks.org/mike/constancy/#key-functions) as the second argument to `selection.data`. The key function returns the key for a given datum or element. For example:

The data is an array of objects, each with a `name` property. The key function can return the associated name:

	var letters = [
		{name: "A", frequency: .08167},
		{name: "B", frequency: .01492},
		{name: "C", frequency: .02780},
  		{name: "D", frequency: .04253},
  		{name: "E", frequency: .12702}
	];

	function name(d) {
		return d.name;
	}

	d3.selectAll("div").data(letters, name);

Again, the selected elements are now bound to data. The elements have also been reordered within the selection to match the data

What happens when there's no matching element for a given datum? Or no matching datum for a given element?

## Enter, Update and Exit ##
When joining elements to data by key there are three possible logicl outcomes:

* *Update*: There was a matching element for a given datum.
* *Enter*: There was no matching element for a given datum.
* *Exit*: There was no matching datum  for a given element.

These are the three selections returned by `selection.data`, `seelction.enter()` and `selection.exit` respectively.

Whgile `update` and `exit` are normal selections, `enter` is a subclass of selection. This is necessary because it represents elements that do not yet exist. An enter selection contains palceholders rather than DOM elements, these palceholders a simply objects with a `__data__` property. The implementation of `enter.select` is then specialized such that nodes are inserted into the group's parent, replacing the placeholder. Thi is why it is critical to call `selection.selectAll` priori to a data join: it establishes the parent node for entering elements.

## Merging Enter  & Update ##



