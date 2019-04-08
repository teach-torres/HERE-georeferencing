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

In the above constructor method, we are accepting an `appId`, an `appCode`, and a `mapElement` 
from the HTML file. Using the token information we can initialize the HERE platform and using the 
HERE platform along with the element information we can display the map. The app id and app code 
tokens can be found in the HERE Developer Portal.

In addition to displaying the map, we are also making it interactive with pan and zoom controls. 
We are initializing the geofencing service and dropping a marker on the map. This marker will 
eventually move around.

With the basics of the project done, open the `index.html` file and include the following in the 
`<script>` tag:

```html
<script>
    const start = async () => {
        const map = new HereMap("APP-ID-HERE", "APP-CODE-HERE", document.getElementById("map"));
    };
    start();
</script>
```

Assuming you're using valid project tokens, the map should show on the screen with a marker 
nearby. This is where things are going to start to get interesting.

## Creating and Uploading Geofence WKT Data to HERE

In a realistic scenario, you're going to want to create your geofence, possibly render it on the 
screen, and upload it to HERE, all without leaving your core project. We also don't want to 
generate WKT files by hand.

There is a solution!

We can actually create any shape we want using the HERE JavaScript SDK, and as part of the SDK we 
can convert those shapes to WKT strings.

Within the index.html file, add the following to your `<script>` tag:
    
```html
<script>
    const start = async () => {
        const map = new HereMap("APP-ID-HERE", "APP-CODE-HERE", document.getElementById("map"));
        const lineString = new H.geo.LineString();
        lineString.pushPoint({ lat: 37, lng: -121 });
        lineString.pushPoint({ lat: 37.2, lng: -121.002 });
        lineString.pushPoint({ lat: 37.2, lng: -121.2 });
        lineString.pushPoint({ lat: 37, lng: -121 });
        const polygon = new H.map.Polygon(lineString);
    };
    start();
</script>
```

Using the above code, we can create a `LineString` which can be used to create a `Polygon` shape. 
To convert the shape into WKT, we can create a function in our `heremap.js` file:

```javascript
polygonToWKT(polygon) {
    const geometry = polygon.getGeometry();
    return geometry.toString();
}
```

If we were to pass our polygon to this function, we would be returned the WKT formatted geometry 
information. Since it is also nice to draw this shape, we could also create a draw function in 
the `heremap.js` file:

```javascript
draw(mapObject) {
    this.map.addObject(mapObject);
}
```

You could move everything into the JavaScript file or everything into the HTML file, it doesn't matter.

With the `polygonToWKT` and `draw` functions available, we can do the following from the `index.html` file:

```javascript
<script>
    const start = async () => {
        const map = new HereMap("APP-ID-HERE", "APP-CODE-HERE", document.getElementById("map"));
        const lineString = new H.geo.LineString();
        lineString.pushPoint({ lat: 37, lng: -121 });
        lineString.pushPoint({ lat: 37.2, lng: -121.002 });
        lineString.pushPoint({ lat: 37.2, lng: -121.2 });
        lineString.pushPoint({ lat: 37, lng: -121 });
        const polygon = new H.map.Polygon(lineString);
        console.log(map.polygonToWKT(polygon));
        map.draw(polygon);
    };
    start();
</script>
```

So we have the WKT information for our shape which should also be visible on our map. Now we 
need to upload that information to the HERE server.

In the project's `heremap.js` file, include the following ^uploadGeofence` function:

```javascript
uploadGeofence(layerId, name, geometry) {
    const zip = new JSZip();
    zip.file("data.wkt", "NAME\tWKT\n" + name + "\t" + geometry);
    return zip.generateAsync({ type:"blob" }).then(content => {
        var formData = new FormData();
        formData.append("zipfile", content);
        return axios.post("https://gfe.api.here.com/2/layers/upload.json", formData, {
            headers: {
                "content-type": "multipart/form-data"
            },
            params: {
                "app_id": this.appId,
                "app_code": this.appCode,
                "layer_id": layerId
            }
        });
    });
}
```

This is probably our most complicated bit of code. We are accepting a `layerId` which can be 
thought of as a WKT id, a name, which can be thought of a geofence name, and the geometry.

The geofence name could be something like GameStop or an actual name representation for the fence. 
Remember, you can have multiple geofences as part of a WKT file, however, ours will only have one.

Using JSZip, we can save our WKT information to a file and add it as a file to the ZIP archive. 
When we want to officially generate the ZIP archive, we can make it a BLOB so we can do an HTTP 
request when it is done.

Using axios, we can take our form data which contains the file and issue an HTTP request to the 
HERE REST API. If all goes smooth, a simple 200 response should be returned.

Going back to the `index.html` file, the following can be added:


```html
<script>
    const start = async () => {
        const map = new HereMap("APP-ID-HERE", "APP-CODE-HERE", document.getElementById("map"));
        const lineString = new H.geo.LineString();
        lineString.pushPoint({ lat: 37, lng: -121 });
        lineString.pushPoint({ lat: 37.2, lng: -121.002 });
        lineString.pushPoint({ lat: 37.2, lng: -121.2 });
        lineString.pushPoint({ lat: 37, lng: -121 });
        const polygon = new H.map.Polygon(lineString);
        console.log(map.polygonToWKT(polygon));
        map.draw(polygon);
        const geofenceResponse = await map.uploadGeofence("1234", "Nic Secret Layer", map.polygonToWKT(polygon));
    };
    start();
</script>
```

You can see that I've given a random id for the layer, provided a name for my geofence, and 
provided the geometry information for my polygon.

As of right now, we should have an application that creates geofences but doesn't make use of 
them.

## Checking if a Position Is Within a Geofence

The next step is to see if any given position is within a geofence. To do this, we'll need to 
provide a layer to check as well as a position. Remember, a layer can have more than one geofence 
which is useful if you have a lot of data that you need to check.

In the `heremap.js` file, include the following function:

```javascript
fenceRequest(layerIds, position) {
    return new Promise((resolve, reject) => {
        this.geofencing.request(
            H.service.extension.geofencing.Service.EntryPoint.SEARCH_PROXIMITY,
            {
                'layer_ids': layerIds,
                'proximity': position.lat + "," + position.lng,
                'key_attributes': ['NAME']
            },
            result => {
                resolve(result);
            }, error => {
                reject(error);
            }
        );
    });
}
```

Given an array of layer ids and a position, we can make a geofence request to see if we're in 
the geofence. Not too difficult in comparison to what we've already accomplished.

To see this `fenceRequest` function in action, let's make some changes to the `constructor` method. 
After creating a marker, add the following:

```javascript
this.map.addEventListener("tap", (ev) => {
    var target = ev.target;
    this.map.removeObject(this.currentPosition);
    this.currentPosition = new H.map.Marker(this.map.screenToGeo(ev.currentPointer.viewportX, ev.currentPointer.viewportY));
    this.map.addObject(this.currentPosition);
    this.fenceRequest(["1234"], this.currentPosition.getPosition()).then(result => {
        if(result.geometries.length > 0) {
            alert("You are within a geofence!")
        } else {
            console.log("Not within a geofence!");
        }
    });
}, false);
```

What we're doing in the above code is we're setting up a listener for tap events on the map. 
If the user taps on the map, we are converting the screen location to geolocation. Using that 
geolocation we can reset the marker and call the `fenceRequest` function. If we get a result, 
we are within the geofence. In the result, you'll have specific information towards which 
geofence was triggered.

This is useful if you need to create geofences for numerous locations based on complex shapes. 

<!--
***********************************************************************************************
-->
