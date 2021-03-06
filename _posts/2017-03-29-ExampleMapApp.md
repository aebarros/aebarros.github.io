---
layout: post
title: Making a Map App using R
subtitle: A step by step guide on how to create a mapping application to display spatial data
published: true
date: '2017-03-29'
---

[Launch  Example Mapping Application](http://aebarros.com/shiny/CatchApp/ExampleMapApp/)

![screenshot of app](/img/examplemapappscreen.png)

[Github link](https://github.com/aebarros/shiny-server/tree/master/ExampleMapApp)

---

## Introduction

This post will act as a step by step guide on how to use R to create basic mapping application for spatial data. This guide assumes a basic understanding of R, [RStudio](https://www.rstudio.com/), and [R Shiny](https://shiny.rstudio.com/).
The guide will be utilizing the following packages:

* [plyr](https://cran.r-project.org/web/packages/plyr/plyr.pdf) 
* [dplyr](https://cran.r-project.org/web/packages/dplyr/dplyr.pdf) 
* [lubridate](https://cran.r-project.org/web/packages/lubridate/lubridate.pdf) 
* [reshape2](https://cran.r-project.org/web/packages/reshape2/reshape2.pdf) 
* [purr](https://cran.r-project.org/web/packages/purrr/purrr.pdf) 
* [shiny](https://cran.r-project.org/web/packages/shiny/shiny.pdf) 
* [shinythemes](https://cran.r-project.org/web/packages/shinythemes/shinythemes.pdf) 
* [leaflet](https://cran.r-project.org/web/packages/leaflet/leaflet.pdf) 
* [rsconnect](https://cran.r-project.org/web/packages/rsconnect/rsconnect.pdf) 

You may use whatever set of data you like along with this guide, as long as it has some key variables:

1. the data you wish to display
2. latitude and longitude coordinates
3. date/time information

We will be utilizing catch information from the California Department of Fish and Wildlifes 20mm net survey. This survey uses a 20mm sled to target juvenile smelt in the Sacramento/San Joaquin delta during the spring months.  
All the scripts and data for this guide can be found [here](https://github.com/aebarros/shiny-server/tree/master/ExampleMapApp).

---

## Collecting Our Data

For this guide I used the 20mm catch data from [CDFW](http://www.dfg.ca.gov/delta/projects.asp?ProjectID=20mm). I downloaded the access database
from [ftp://ftp.dfg.ca.gov/Delta%20Smelt/](ftp://ftp.dfg.ca.gov/Delta%20Smelt/) labeled as "20-mm.mdb". The 20 mm data base has four tables that we will be using:

* 20 mm stations (contains a list of station names and gps coordinates)
* Catch (tells us the catch of each species of fish for each tow using a fishcode, this is what we wish to display)
* Fish Codes (links the fishcodes to common names for each species)
* Tow info (links the catch info to a specific tow, station, date, and duration)

I exported each of these tables as a tab delimited .txt file into a "data" folder within my R-project.

---

## Cleaning Our Data

In order to make the mapping application I need to bring together all the data from the above tables and get it into a format that I 
can work with. My end goal here is a dataset similar to the following:

Station|Date|longitude|latitude|Common.Name|CPUE
---|---|---|---|---|---
809|1995-04-24|-121.6892|38.05250|American shad|0

(note: catch per unit effort (CPUE) is the data we will be displaying)

In order to reach this point we will be creating a cleaning script titled "global.R" saved within our project.
We begin the script by loading our packages:

```R
library(plyr)
library(dplyr)
library(lubridate)
library(reshape2)
library(purrr)
```

followed by loading the required data:

```R
data.catch=read.table("data/catch.txt", header= T, sep= "\t")
fishcodes=read.table("data/fishcodes.txt", header= T, sep= "\t")
data.stations=read.table("data/stations.txt", header= T, sep= "\t")
data.tows=read.table("data/towinfo.txt", header= T, sep= "\t")
```

Next I want to inspect the elements using the "sapply" funtion. This funtion is useful to discover the class/type of each vector.

```R
#####inspect elements#
head(data.stations)
sapply(data.stations,class)
```

After running this code you may notice some obvious issues with our data:

1. Latitude and Longitude are each broken up into seperate vectors for degrees, minutes, and seconds
2. Each of those vectors has a different class, either integer or numeric

This is not at all what we want, so our first step will be to turn our Latitude and Longitude into decimal degree format, and each under its own vector using the following calls:

```R
#transform into decimal degrees
data.stations$LatS<-data.stations$LatS/3600
data.stations$LatM<-data.stations$LatM/60
data.stations$LonS<-data.stations$LonS/3600
data.stations$LonM<-data.stations$LonM/60
#add minutes and seconds together
data.stations$LatMS<-data.stations$LatM+data.stations$LatS
data.stations$LonMS<-data.stations$LonM+data.stations$LonS
#combine data
data.stations$Latitude <- paste(data.stations$LatD,data.stations$LatMS)
data.stations$Longitude <- paste(data.stations$LonD,data.stations$LonMS)
head(data.stations)
#remove spaces and first zero, replace with "."
data.stations$Latitude <- gsub(" 0.", ".", data.stations$Latitude)
data.stations$Longitude <- gsub(" 0.", ".", data.stations$Longitude)
head(data.stations)
#add "-" infront of all longitude
data.stations$negative <- rep("-",nrow(data.stations)) # make new column 
data.stations$Longitude<-paste(data.stations$negative, data.stations$Longitude)
data.stations$Longitude <- gsub(" ", "", data.stations$Longitude)
head(data.stations)
#keep columns we need#
keep<-c("Station","Latitude","Longitude")
data.stations<-data.stations[ , which(names(data.stations) %in% keep)]
head(data.stations)
#transform Lat and Long to numeric class
data.stations<-transform(data.stations, Latitude = as.numeric(Latitude), Longitude = as.numeric(Longitude))
```

Now a simple `head(data.stations)` call will show us that at least our stations information is formated properly.

Our next step in preparing our data for the application is to join all our tables together using a series of `inner_join` calls. I do this here with dplyr piping "%>%".

```R
data<-data.catch%>%
  inner_join(data.stations)%>%
  inner_join(data.tows)%>%
  inner_join(fishcodes)
```

Using `head(data)` and `sapply(data,class)` calls we can see that the joined data table has a load of vectors we don't need, as well as
 the Date vector having a class of "factor". We need the Date vector to have a "date" class for use by the shiny app, and to remove the unnecessary columns like so:
 
```R
#format dates#
sapply(data,class)
data$Date <- as.Date(data$Date , "%m/%d/%Y")

#keep columns we need#
keeps<-c("Date","Station","Latitude","Longitude","Fish.Code","Catch","Duration","Common.Name")
data<-data[ , which(names(data) %in% keeps)]
```

Next I want to do two things, first calculate a Catch Per Unit of Effort (here calculated as catch/tow duration) and then add in an "All" option which combines the catch for all species caught in each tow. This is all done using the following:

```R
#calculate CPUE
data$CPUE=data$Catch/data$Duration
head(data)

###Next section to calculate "All" CPUE for each tow
#reshape fish catch to wide format
datawide <- dcast(data, Station +Latitude+Longitude+Duration+ Date ~ Common.Name, value.var="CPUE",fun.aggregate=sum)
head(datawide)
#learn which column numbers apply to fish
which(colnames(datawide)=="American shad")
which(colnames(datawide)=="yellowfin goby")
#calculate All
datawide$All=rowSums(datawide[,6:71], na.rm=T)
head(datawide)
#melt back straight
data=melt(datawide,id.vars=c("Station","Date","Longitude","Latitude"), measure.vars=c(6:72),
             variable.name="Common.Name",
             value.name="CPUE")
```

The dcast and melt calls used above are both from the reshape2 package, and restructure a molten data frame into an array or data frame.

Finally we want to change the vector name for the Latitude and Longitude vectors in our data frame. When creating the application, the leaflet package will automatically recognize the lat and long vectors as coordinates to use, but not if they are named "Latitude" and "Longitude" with capital "L"s. Here we simply change those to lower case to make life easier down the road.

```R
#rename Lat and Lon for leaflet
data<-rename(data, latitude=Latitude, longitude=Longitude)
```

Now our data is clean and ready to use!

---

## Creating the Application

Next comes the fun part, getting to create our application! This section will be split in two, starting with a section covering the ui, and finishing with the server. If none of that make sense to you I suggest visiting [this page](https://shiny.rstudio.com/tutorial/)

Before anything else, create a new script in your project named "app.R". This will be recognized as your application when you later go to deploy.

### UI

The first section of our script is the ui, which basically sets out the user interface. Here we establish the html layout for our app, and set up the inputs to take values from the user. Once again we start the script by loading the required packages, and then we will source "global.R" that we created in the data cleaning section.

```R
###Load Packages###
##always load plyr before dplyr##
library(plyr)
library(dplyr)
library(lubridate)
library(shiny)
library(reshape2)
library(leaflet)
library(shinythemes)
library(rsconnect)

##source the global file with all the necessary data
source("global.R")
```

Next follows the ui code itself. A few things of notes with this:

*  Shiny themes are community created layouts provided by the shinythemes package that effect the format of the page. Examples of those themes can be found [here](https://rstudio.github.io/shinythemes/).
* I've set the height of the map itself to be 100% of the users screen height minus 120 pixels using `tags$style(type = "text/css", "#map {height: calc(100vh - 120px) !important;}"),`
* The `apsolutePanel` sets up a panel with all our input select options on the right side of the main panel. The header is set as `h2("Catch Explorer")`
* Our two input selections are named "Species" and "Date range", these allow the user to select the parameters of the map that will be displayed. With the species input I use `selectizeInput()` which allows for selecting from a scroll bar or typing in the species name. For the date range input I used a `dateRangeInput` which allows the user to select a range of dates from which tow data will be displayed.
* The `submitButton("Submit"))` at the bottom of the ui allows the user to select inputs and then submit the changes. Without this option the map will automatically update each time a new parameter is selected, which can be problematic when setting the date range.

```R
##UI building
ui = bootstrapPage(theme = shinytheme("sandstone"),
                     div(
                       id = "app-content",
                       navbarPage("CDFW 20mm Catch"),
                       tags$style(type = "text/css", "#map {height: calc(100vh - 120px) !important;}"),
                       leafletOutput("map", height = "100%"),
                       tags$div(id="cite",
                                'Data compiled by Arthur Barros (2017).'
                       ),
                       absolutePanel(id = "controls", class = "panel panel-default", fixed = TRUE,
                                     draggable = FALSE, top = 60, left = "auto", right = 20, bottom = "auto",
                                     width = 200, height = "auto",
                                     h2("Catch Explorer"),
                                     selectizeInput("species", "Species",
                                                    unique(as.character(data$Common.Name))),
                                     dateRangeInput("dates","Date range", start="2015-03-01", end="2015-03-31"),
                                     textOutput("DateRange"),
                                     submitButton("Submit"))
                     )
)
```

---

### Server

Now let's take a look at the server side of the script:

```R
server <- function(input, output, session) {
  ###########Interactive Map##############################################
  #Reactive expression used to filter out by user selected variables for final data "filtered()"
  #for map generation
  filtered<-reactive({
    filtered.species<- data[data$Common.Name==input$species,]
    filtered.dates<-filtered.species[filtered.species$Date>=input$dates[1] & filtered.species$Date<=input$dates[2],]
    ##take averages of data in date range
    filtered.dates=filtered.dates%>%
      group_by_("Station","longitude","latitude","Common.Name")%>%
      summarise(CPUE=mean(CPUE,na.rm=TRUE))
    filtered.date <-  filtered.dates %>% mutate(CPUE = replace(CPUE,CPUE==0,NA))
  })
  
  #produces base map
  mymap1 <- reactive({
    leaflet(filtered())%>% addProviderTiles("Hydda.Full")%>%
      fitBounds(~min(longitude)-.005, ~min(latitude)-.005, ~max(longitude)+.005, ~max(latitude)+.005)
  })
  #map2 is used if no data fits selected parameters, and will be blank
  mymap2 <- reactive({
    leaflet()%>% addProviderTiles("Hydda.Full")%>%
      setView(lng = -122.40, lat = 37.85, zoom = 9)
  })
  #renders map for main panel in ui using an if function to create map fitting parameters
  output$map <- renderLeaflet({
    if(nrow(filtered())==0){mymap2()}else{mymap1()}
  })
  
  ##Provides date range count
  output$DateRange <- renderText({
    paste("Your date range is", 
          difftime(input$dates[2], input$dates[1], units="days"),
          "days")
  })
  
  ##this function used to be an observation, switched to function
  ##in order to allow for downloading in rmarkdown
  myfun <- function(map){
    #this "pal" produces the desired colors and bins for distinguishing CPUE
    pal<-colorBin(
      palette="Reds",
      domain=filtered()$CPUE,
      bins=c(0,.1,1,10,100,1000),
      pretty = TRUE,
      na.color="black")
    
    #next call populates map with markers based on filtered() data
    addCircleMarkers(map, data = filtered(),lng = ~longitude, lat = ~latitude,radius=~ifelse(is.na(filtered()$CPUE),2,10),
                     stroke=TRUE, color="black",weight=2, fillOpacity=1,
                     fillColor=~pal(filtered()$CPUE),
                     popup = ~paste("Catch per Minute Towed:", filtered()$CPUE, "<br>",
                                    "Station:", Station,"<br>",
                                    "Coordinates:", latitude,",",longitude,"<br>"))%>%
      addLegend("bottomleft", pal=pal, values=filtered()$CPUE, title="Catch Per Minute of Tow",
                opacity=1)
  }
  #here is an observation using leafletProxy to take the above function and run it on our map
  #also have to have the "clear" calls here, as they won't work in the myfun
  observe({
    leafletProxy("map")%>%
      clearControls%>%
      clearMarkers()%>% myfun()
  })
  
  # Stop shiny app when closing the browser
  session$onSessionEnded(stopApp)
}

shinyApp(ui = ui, server = server)
```

This is a lot to go through, so I'm going to try and break it into bite size chunks.
The first section uses a reactive expression `reactive({ })` to build a new reactive data frame based on the user selections.  
* The first bit creates a new df based on the species input `filtered.species<- data[data$Common.Name==input$species,]` that only includes data with the selected species.
* The second bit `filtered.dates<-filtered.species[filtered.species$Date>=input$dates[1] & filtered.species$Date<=input$dates[2],]` does the same but based on the user selected date range.
* Finally we wish to average all the CPUE data for the species within the selected date range. This is done with a `group_by` and `summarise` call. We also want to change those rows with a 0 for CPUE to a NA, so as to aid in mapping later on.
* The complete data frame is named "filtered", and will be referenced as `filtered()` throughout the rest of the script.


```R
#Reactive expression used to filter out by user selected variables for final data "filtered()"
  #for map generation
  filtered<-reactive({
    filtered.species<- data[data$Common.Name==input$species,]
    filtered.dates<-filtered.species[filtered.species$Date>=input$dates[1] & filtered.species$Date<=input$dates[2],]
    ##take averages of data in date range
    filtered.dates=filtered.dates%>%
      group_by_("Station","longitude","latitude","Common.Name")%>%
      summarise(CPUE=mean(CPUE,na.rm=TRUE))
    filtered.date <-  filtered.dates %>% mutate(CPUE = replace(CPUE,CPUE==0,NA))
  })
  ```
  
The next section produces two base maps for us to use. This is the first time we use the leaflet package. The base map used is titled "Hydda.Full" and other provider tiles can be seen [here](https://leaflet-extras.github.io/leaflet-providers/preview/).  
* The first map `mymap1` uses a reactive expression to set the base map with boundaries set by the max and min latitude and longitude from the filtered() df.
* The second map `mymap2` is used when no data fits the selected parameters (for example if a date range is selected during which no sampling tows occured) and just shows a blank map of the bay area.
* The third section is an "if" statement to render the map, making the app display map2 if the number of rows in the filtered() df is equal to 0, meaning that no data fits those parameters. If `nrows(filtered())!=0` then map1 will display.
  
  ```R
    #produces base map
  mymap1 <- reactive({
    leaflet(filtered())%>% addProviderTiles("Hydda.Full")%>%
      fitBounds(~min(longitude)-.005, ~min(latitude)-.005, ~max(longitude)+.005, ~max(latitude)+.005)
  })
  #map2 is used if no data fits selected parameters, and will be blank
  mymap2 <- reactive({
    leaflet()%>% addProviderTiles("Hydda.Full")%>%
      setView(lng = -122.40, lat = 37.85, zoom = 9)
  })
  #renders map for main panel in ui using an if function to create map fitting parameters
  output$map <- renderLeaflet({
    if(nrow(filtered())==0){mymap2()}else{mymap1()}
  })
  ```
  
The final part of our application will be used to populate our map with our selected data. It is only using data from the filtered() df at this point.  
* The `pal<-` call sets the pallet for CPUE values. Basically it uses default shades of red, to assign color values to each marker that will be displayed on the map based on the CPUE value. For any station with a CPUE value of NA, the color will be black.
* The `addCircleMarkers` is a leaflet call to populate our map with markers.
  * Using an `ifelse` call I have set the radius size for each marker to be a default of 10, or 2 if the CPUE is NA.
  * The `fillColor` call sets the color of each marker using the pallet set above, again based on the CPUE value.
  * The `popup` call allows a text box to display when the user clicks on a marker. In this case the text box will display the CPUE, station number, and its coordinates.
  * The `addLegend` produces a map legend based on the pallet.
* The last function is an `observe({ })` function. This clears any markers and legend on the map before producing a new one, each time the submit button is clicked.
  
  ```R
  myfun <- function(map){
    #this "pal" produces the desired colors and bins for distinguishing CPUE
    pal<-colorBin(
      palette="Reds",
      domain=filtered()$CPUE,
      bins=c(0,.1,1,10,100,1000),
      pretty = TRUE,
      na.color="black")
    
    #next call populates map with markers based on filtered() data
    addCircleMarkers(map, data = filtered(),lng = ~longitude, lat = ~latitude,radius=~ifelse(is.na(filtered()$CPUE),2,10),
                     stroke=TRUE, color="black",weight=2, fillOpacity=1,
                     fillColor=~pal(filtered()$CPUE),
                     popup = ~paste("Catch per Minute Towed:", filtered()$CPUE, "<br>",
                                    "Station:", Station,"<br>",
                                    "Coordinates:", latitude,",",longitude,"<br>"))%>%
      addLegend("bottomleft", pal=pal, values=filtered()$CPUE, title="Catch Per Minute of Tow",
                opacity=1)
  }
  #here is an observation using leafletProxy to take the above function and run it on our map
  #also have to have the "clear" calls here, as they won't work in the myfun
  observe({
    leafletProxy("map")%>%
      clearControls%>%
      clearMarkers()%>% myfun()
  })
  ```
  
Finaly we wrap it all up with a session end script, which allows the application to shut itself down when the window is closed. This is useful specifically when hosting the ap online, so as not to drain server resources.
  
  ```R
    # Stop shiny app when closing the browser
  session$onSessionEnded(stopApp)
}

shinyApp(ui = ui, server = server)
```
  
  Now our app is complete! Run it using the "Run app" button (if you are using RStudio) or the runapp() call.
