# article1

## NYT Interactive on Strikeouts

\`\`\`{r echo = F, message = F, cache = F} library\(knitr\) opts\_chunk$set\(results = 'asis', comment = NA, message = F, tidy = F\) require\(rCharts\) options\(RCHART\_WIDTH = 600, RCHART\_HEIGHT = 400\)

```text
# Replicating NY Times Interactive Graphic

This tutorial explains in detail, how I used `rCharts` to replicate this NY times [interactive graphic](http://www.nytimes.com/interactive/2013/03/29/sports/baseball/Strikeouts-Are-Still-Soaring.html?ref=baseball) on strikeouts in baseball. The end result can be seen [here](http://glimmer.rstudio.com/ramnathv/strikeouts) as a `shiny` application.

### Data

The first step is to get data on strikeouts by team across years. The NY Times graphic uses data scraped from [baseball-reference](http://www.baseball-reference.com/), using the `XML` package in R. However, I will be using data from the R package [Lahman](http://cran.r-project.org/web/packages/Lahman/index.html), which provides tables from [Sean Lahman's Baseball Database](http://www.seanlahman.com/baseball-archive/statistics/) as a set of data frames.

The data processing step involves using the [plyr](http://cran.r-project.org/web/packages/plyr/index.html) package to create two data frames:

1. `team_data` containing `SOG` (strikeouts per game) by `yearID` and team `name`
2. `league_data` containing `SOG` by `yearID` averaged across the league.


```{r}
require(Lahman)
require(plyr)
dat = Teams[,c('yearID', 'name', 'G', 'SO')]
team_data = na.omit(transform(dat, SOG = round(SO/G, 2)))
league_data = ddply(team_data, .(yearID), summarize, SOG = mean(SOG))
```

## Charts

We will start by first creating a scatterplot of `SOG` by `yearID` across all teams. We use the `rPlot` function which uses the PolyChartsJS library to create interactive visualizations. The formula interface specifies the x and y variables, the data to use and the type of plot. We also specify a `size` and `color` argument to style the points. Finally, we pass a `tooltip` argument, which is a javascript function that overrides the default tooltip to display the information we require. You will see below the R code and the resulting chart.

```text
require(rCharts)
p1 <- rPlot(SOG ~ yearID, data = team_data, type = 'point', 
  size = list(const = 2), color = list(const = '#888'), 
  tooltip="#! function(item){return item.SOG + ' ' + item.name + ' ' + item.yearID} !#"
)
p1
```

Now, we need to add a line plot of the average `SOG` for the league by `yearID`. We do this by adding a second layer to the chart, which copies the elements of the previous layer and overrides the `data`, `type`, `color` and `tooltip` arguments. The R code is shown below and you will note that the resulting chart now shows a blue line chart corresponding to the league average `SOG`.

```text
p1$layer(data = league_data, type = 'line', 
  color = list(const = 'blue'), copy_layer = T, tooltip = NULL)
p1
```

Finally, we will overlay a line plot of `SOG` by `yearID` for a specific team `name`. Later, while building the shiny app, we will turn this into an input variable that a user can choose from a dropdown menu. We use the layer approach used earlier and this time override the `data` and `color` arguments so that the line plot for the team stands out from the league average.

```text
myteam = "Boston Red Sox"
p1$layer(data = team_data[team_data$name == myteam,], color = list(const = 'red'),
  copy_layer = T)
p1
```

Let us add a little more interactivity to the chart. To keep it simple, we will use handlers in PolychartJS to initiate an action when a user clicks on a point. Here is where the magic of `knitr` shines \(thanks to [yihui/knitr](http://github.com/yihui/knitr)\), as we can mix coffeescript code in our document. The current handler is a simple one, which just displays the name of the team clicked on. If you are familiar with Coffeescript, the code is self explanatory.

\`\`\`{r interactivity, engine='coffee', echo = T, eval = F} graph\_chart3.addHandler \(type, e\) -&gt; data = e.evtData if type == 'click' alert "You clicked on the team: " + data.name.in\[0\]

```text
### Application

Now it is time to convert this into a Shiny App. We will throw the data processing code into `global.R` so that it can be accessed both by `ui.R` and `server.R`. For the dropdown menu allowing users to choose a specific team, we will restrict the choices to only those which have data for more than 30 years. Accordingly, we have the following `global.R`.

```{r}
## global.R
require(Lahman)
require(plyr)
dat = Teams[,c('yearID', 'name', 'G', 'SO')]
team_data = na.omit(transform(dat, SOG = round(SO/G, 2)))
league_data = ddply(team_data, .(yearID), summarize, SOG = mean(SOG))
THRESHOLD = 30
team_appearances = count(team_data, .(name))
teams_in_menu = subset(team_appearances, freq > THRESHOLD)$name
```

For the UI, we will use a bootstrap page with controls being displayed in the sidebar. Shiny makes it really easy to create a page like this. See the annotated graphic below and the `ui.R` code that accompanies it to understand how the different pieces fit together.

 [![](http://www.clipular.com/c?5151018=o2zGtsIzD20s1dp25X1mLRSUTMk&f=e04b448074961bf4efcb426b91886d8b)](http://glimmer.rstudio.com/ramnathv/strikeouts)

We now need to write the server part of the shiny app. Thankfully, this is the easiest part, since it just involves wrapping the charting code inside `renderChart` and replacing user inputs to enable reactivity. We add a few more lines of code to set the height and title and remove the axis titles, since they are self explanatory.

