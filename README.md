# Creating your first satellite image

## http://bit.ly/nicar19-landsat

In this hands-on training, you'll learn how to find, download, combine, and turn data captured by satellites into ready-for-publication images. You'll learn three ways to do this by using point and click tools, command line and Google Earth Engine.

[Best tutorial on using photoshop to process Landsat](https://earthobservatory.nasa.gov/blogs/elegantfigures/2013/10/22/how-to-make-a-true-color-landsat-8-image/)

[Best tutorial on using landsat util to process Landsat](https://www.developmentseed.org/blog/2014/08/29/landsat-util/)

## First things first

Request access to Google Earth Engine code environment

https://signup.earthengine.google.com/#!/

The machines we used already had the following required software installed:
* [Landsat Util](https://pythonhosted.org/landsat-util/installation.html)
* [GIMP](https://www.gimp.org/)

The GIMP process can be achived in Photoshop as well, the method is similar. I find the Photoshop way easier.

## Where does this data come from

We didn't do this in the session but these tools provide various ways to download Landsat scenes. I think Libra is the easiest to use.

* [Libra from Development Seed](https://libra.developmentseed.org)
* [Landsat Util](https://pythonhosted.org/landsat-util/)
* [Earth Explorer](https://earthexplorer.usgs.gov/)

## Point and click with GIMP

1. Open the three images in GIMP
	* You want the XXX_B2.tif XXX_B3.tif and XXX_B4.tif files 

2. Open the Compose panel under `Colors > Components > Compose`

3. Select "RGB" in Color Model

4. Specify your channels 
	* Red: X_B4.tif
	* Green: X_B3.tif
	* Blue: X_B2.tif

	Click OK

6. Your image is really dark. Lighten it up by going to `Colors > Levels`
	take the white carrot and drag it to the edge of the histogram

7. From the set of eyedropper buttons on the bottom right of the panel, select the middle one, "Pick gray point"

8. Zoom into your image and find a cloud or white roofed building, click it. Click "OK" in the Levels panel

9. Open the Curves panel `Colors > Curves`

9. In panel, select each channel from the Channel drop down, and drag the dot on the bottom left for each to the point on the histogram where the curve gets steep.

10. Switch back to the Value view in the histogram and pull the middle of the black line up. Click OK

Using this method will remove all of the geographic metadata from your image. [Here is a way to get it back](https://gis.stackexchange.com/a/108703) using the command line tool GDAL.

## On the command line with landsat-util

1. Open up the command line and navigate to the project folder

2. run `landsat process LC08_L1TP_041036_20190122_20190122_01_RT --pansharpen --bands 432`

3. the output tells you where your file is located typically in your home directory inside of a folder at `landsat/processed/{landsat ID}`


## Using Google Earth Engine

1. Navigate to https://code.earthengine.google.com/

2. Create a new script

3. Pick a point you want to focus on

4. Define your parameters

```
var o = {
  start: ee.Date('2013-01-01'),
  finish: ee.Date('2019-03-09'),
  target: geometry,
  cloud_cover_lt: 0.8,
  bands:["B4", "B3", "B2"]
}
```

5. Load all of the landsat scenes

```
var filteredCollection = ee.ImageCollection("LANDSAT/LC08/C01/T1_RT")
```

6. Filter to the one that is over your point

```javascript
var filteredCollection = ee.ImageCollection("LANDSAT/LC08/C01/T1_RT")
  .filterBounds(o.target)

```


7. Limit to those that have low cloud coverage

```javascript
var filteredCollection = ee.ImageCollection("LANDSAT/LC08/C01/T1_RT")
  .filterBounds(o.target)
  .filterMetadata("CLOUD_COVER", "less_than", o.cloud_cover_lt)
```

8. Limit to captures of during daylight

```javascript
var filteredCollection = ee.ImageCollection("LANDSAT/LC08/C01/T1_RT")
  .filterBounds(o.target)
  .filterMetadata("CLOUD_COVER", "less_than", o.cloud_cover_lt)
  .filterMetadata("SUN_ELEVATION", "greater_than", 0)
```

**Code Check** 
Right now your script should look like this
```javascript
var o = {
  start: ee.Date('2013-01-01'),
  finish: ee.Date('2019-03-09'),
  target: geometry,
  cloud_cover_lt: 0.8,
  bands:["B4", "B3", "B2"]
}

var filteredCollection = ee.ImageCollection("LANDSAT/LC08/C01/T1_RT")
  .filterBounds(o.target)
  .filterMetadata("CLOUD_COVER", "less_than", o.cloud_cover_lt)
  .filterMetadata("SUN_ELEVATION", "greater_than", 0)
```

9. Pick the most recent of those

```javascript
var scene = ee.Image(filteredCollection.sort("DATE_ACQUIRED", false).first());
```

10. Set your parameters based on the data

You can do it this way by manually defining the extent of your data

```javascript
var params = {
  bands: o.bands,
  max: [10000,10000,10000],
  min: [1000,1000,1000]
}

```


More complexly you can try to do it automatically by programatically calculating the extent of the data in your target area

```javascript
var bandMax = filteredCollection.median()
    .reduceRegion({
      geometry: o.target,
      reducer: ee.Reducer.percentile([99]),
      scale: 60,
      tileScale: 16
    })
    
  var bandMin = filteredCollection.median()
    .reduceRegion({
      geometry: o.target,
      reducer: ee.Reducer.percentile([1]),
      scale: 60,
      tileScale: 16
    })


var params = {
  bands: o.bands,
  max: bandMax.values(o.bands).getInfo(),
  min: bandMin.values(o.bands).getInfo()
}

```

11. Plot your scene in the environment

```javascript
Map.addLayer(scene, params)
```

12. Save to your google drive as a jpeg

```javascript
var export_image = scene.visualize(params)

Export.image.toDrive({
    image: export_image,
    description: "my_scene_from_nicar",
    scale: 30,
    maxPixels: 240000000000
})
```

13. Click the "Run" button to exicute your code. In the "Tasks" panel clikc "Run" on the item created to begin the image export process.

Exporting images from Earth Engine can be slow. To speed it up, try drawing a shape on the map viewer and using it to crop with. Then update the export section to look a like this.

```javascript
var export_image = scene.visualize(params)

Export.image.toDrive({
    image: export_image,
    description: "my_scene_from_nicar",
    scale: 30,
    region: geometry, #make sure this matches the name of whatever shape you drew 
    maxPixels: 240000000000
})
```


## Now what?

These are some stories that have used Landsat imagery to various ends. Some simply make use of true-color images like we made here. Some combine bands that capture reflections that are invisible to humans to detect vegetation health or highlight land use.

* [The island Bangladesh is thinking of putting refugees on is hardly an island at all](https://qz.com/1075444/the-island-bangladesh-is-thinking-of-putting-refugees-is-hardly-an-island-at-all/) by Quartz
* [Who is the Wet Prince of Bel Air? Here are the likely culprits](https://www.revealnews.org/article/who-is-the-wet-prince-of-bel-air-here-are-the-likely-culprits/) by Reveal
* [Welcome to Fabulous Las Vegas: While supplies last](https://projects.propublica.org/las-vegas-growth-map/) by Propublica
* [A Rogue State Along Two Rivers](https://www.nytimes.com/interactive/2014/07/03/world/middleeast/syria-iraq-isis-rogue-state-along-two-rivers.html) by The New York Times

