# HERE-georeferencing

The idea is that we're going to draw a shape on the map and upload the geometry information for 
that shape to HERE. Then when we move the marker around on the map, we're going to ask HERE if 
the marker falls in the geometry that we had uploaded. This is a simple example where we only 
have one geometry, but the uploaded information could easily have much more, making it easier 
than checking the boundary information by hand.

So what exactly is happening with the geofence and what do we need to upload to HERE? To start, 
we need to come up with a series of points that create a polygon of any complexity.

Take the following points:

```terminal
[

    { lat: 37, lng: -121 },

    { lat: 37.2, lng: -121.002 },

    { lat: 37.2, lng: -121.2 },

    { lat: 37, lng: -121 },

]
```

The above points will create a triangle. Wait, don't triangles only have three points? Yes, but 
because we're creating a shape, we need to close the loop and bring the final vertex home. The 
first point and the final point are the same.

So this is great, but an array of points of kind of useless to the HERE server. There is actually 
a format that must be followed which extends beyond HERE. We need to be supplying Well-Known 
Text (WKT) formatted data.

If we wanted to convert our array of points to WKT, we can do something like this:

```
POLYGON ((-121 37,-121.002 37.2,-121.2 37.2,-121 37))
```

Yes, in the above example the longitude comes first, but that is fine. The above POLYGON is a 
start, but it doesn't complete the story for us. We need to use it correctly in a file. Take the 
following for example:

```
NAME        WKT
Triangle    POLYGON ((-121 37,-121.002 37.2,-121.2 37.2,-121 37))
```

The above would be valid WKT file data. We have a name for our fence, which we're randomly 
calling Triangle, and we have our shape. We could have any number of names and shapes in this 
file separated by new lines. However, it is important that this is a tab delimited file. 
Delimiting by space characters will not work.

Now that you have a brief understanding of geofencing and WKT files, we can start the development 
of a project.

## Developing a JavaScript Project With HERE

To keep things simple, it is best we start by creating a new project. Once we have the foundation 
in place, we can add complexity with the geofence and WTK components.

Somewhere on your computer, create the following two files:

```terminal
index.html
heremap.js
```

All of our HTML will go in the `index.html` file and most of the JavaScript will go in the `heremap.js` 
file. There will also be JavaScript in the `index.html` file, but you're free to optimize your 
project however is best.

Open the index.html file and include the following:

```html
<html>

    <body style="margin: 0">
        <div id="map" style="width: 100vw; height: 100vh"></div>
        <script src="http://js.api.here.com/v3/3.0/mapsjs-core.js" type="text/javascript" charset="utf-8"></script>
        <script src="http://js.api.here.com/v3/3.0/mapsjs-service.js" type="text/javascript" charset="utf-8"></script>
        <script src="http://js.api.here.com/v3/3.0/mapsjs-mapevents.js" type="text/javascript" charset="utf-8"></script>
        <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
        <script src="https://stuk.github.io/jszip/dist/jszip.js" type="text/javascript" charset="utf-8"></script>
        <script src="heremap.js"></script>
        <script>

            // JavaScript here...

        </script>
    </body>
</html>
```

<!---
In computer programming, boilerplate code or just boilerplate are sections of code that have 
to be included in many places with little or no alteration.
-->

In the above boilerplate code, we are creating an HTML placeholder element for our map. This 
placeholder element is identified with the id property. We are also importing several JavaScript 
libraries, three of which are part of the HERE JavaScript SDK. In addition to the HERE JavaScript 
SDK, we are also including axios for HTTP requests and JSZip for archiving our WKT file data.

To send our WKT information, it must be done via HTTP and the WKT information must exist as part 
of a ZIP archive. There is no wrong way to do this, but I personally find it more convenient if 
we can create our geofences directly in our project, hence the two libraries.

Now open the project's `heremap.js` file and include the following:

```javascript
class HereMap {
    constructor(appId, appCode, mapElement) { }
    draw(mapObject) { }
    polygonToWKT(polygon) { }
    uploadGeofence(layerId, name, geometry) { }
    fenceRequest(layerIds, position) { }
}
```
***********************************************************************************************
You can see we have a `HereMap` class with several functions, including a constructor method. Most 
of the heavy lifting will be accomplished in this class.

Let's start configuring our map within the `constructor` method:

```javascript
constructor(appId, appCode, mapElement) {
    this.appId = appId;
    this.appCode = appCode;
    this.platform = new H.service.Platform({
        "app_id": this.appId,
        "app_code": this.appCode
    });
    this.map = new H.Map(
        mapElement,
        this.platform.createDefaultLayers().normal.map,
        {
            zoom: 10,
            center: { lat: 37, lng: -121 }
        }
    );
    const mapEvent = new H.mapevents.MapEvents(this.map);
    const behavior = new H.mapevents.Behavior(mapEvent);
    this.geofencing = this.platform.getGeofencingService();
    this.currentPosition = new H.map.Marker({ lat: 37.21, lng: -121.21 });
    this.map.addObject(this.currentPosition);
}
```

