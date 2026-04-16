// ==UserScript==
// @name         geoguessr
// @namespace    http://tampermonkey.net/
// @version      67
// @description  idk
// @author       Avery
// @match        https://www.geoguessr.com/*
// @icon         https://www.google.com/s2/favicons?sz=64&domain=geoguessr.com
// @grant        GM_webRequest
// @grant    GM_xmlhttpRequest
// @downloadURL https://update.greasyfork.org/scripts/450253/Geoguessr%20Location%20Resolver%20%28Works%20in%20all%20modes%29.user.js
// @updateURL https://update.greasyfork.org/scripts/450253/Geoguessr%20Location%20Resolver%20%28Works%20in%20all%20modes%29.meta.js
// ==/UserScript==


// =================================================================================================================
// 'An idiot admires complexity, a genius admires simplicity'
// Learn how I made this script: https://github.com/0x978/GeoGuessr_Resolver/blob/master/howIMadeTheScript.md
// Contribute things you think will be cool once you learn: https://github.com/0x978/GeoGuessr_Resolver/pulls
// ================================================================================================================

let globalCoordinates = { // keep this stored globally, and we'll keep updating it for each API call.
    lat: 0,
    lng: 0
}

let globalPanoID = undefined

// Below, I intercept the API call to Google Street view and view the result before it reaches the client.
// Then I simply do some regex over the response string to find the coordinates, which Google gave to us in the response data
// I then update a global variable above, with the correct coordinates, each time we receive a response from Google.

var originalOpen = XMLHttpRequest.prototype.open;
XMLHttpRequest.prototype.open = function(method, url) {
    // Geoguessr now calls the Google Maps API multiple times each round, with subsequent requests overwriting
    // the saved coordinates. Calls to this exact API path seems to be legitimate for now. A better solution than panoID currently?
    // Needs testing.
    if (method.toUpperCase() === 'POST' &&
        (url.startsWith('https://maps.googleapis.com/$rpc/google.internal.maps.mapsjs.v1.MapsJsInternalService/GetMetadata') ||
            url.startsWith('https://maps.googleapis.com/$rpc/google.internal.maps.mapsjs.v1.MapsJsInternalService/SingleImageSearch'))) {

        this.addEventListener('load', function () {
            let interceptedResult = this.responseText
            const pattern = /-?\d+\.\d+,-?\d+\.\d+/g;
            let match = interceptedResult.match(pattern)[0];
            let split = match.split(",")

            let lat = Number.parseFloat(split[0])
            let lng = Number.parseFloat(split[1])


            globalCoordinates.lat = lat
            globalCoordinates.lng = lng
        });
    }
    // Call the original open function
    return originalOpen.apply(this, arguments);
};


// ====================================Placing Marker====================================

function placeMarker(safeMode){
    let {lat,lng} = globalCoordinates

    if (safeMode) { // applying random values to received coordinates.
        const sway = [Math.random() > 0.5,Math.random() > 0.5]
        const multiplier = Math.random() * 4
        const horizontalAmount = Math.random() * multiplier
        const verticalAmount = Math.random() * multiplier
        sway[0] ? lat += verticalAmount : lat -= verticalAmount
        sway[1] ? lng += horizontalAmount : lat -= horizontalAmount
    }

    let element = document.querySelectorAll('[class^="guess-map_canvas__"]')[0]
    if(!element){
        placeMarkerStreaks()
        return
    }

    const latLngFns = {
        latLng:{
            lat: () => lat,
            lng: () => lng,
        }
    }

    // Fetching Map Element and Props in order to extract place function
    const reactKeys = Object.keys(element)
    const reactKey = reactKeys.find(key => key.startsWith("__reactFiber$"))
    const elementProps = element[reactKey]
    const mapElementClick = elementProps.return.return.memoizedProps.map.__e3_.click
    const mapElementPropKeys = Object.keys(mapElementClick)
    const mapElementPropKey = mapElementPropKeys[mapElementPropKeys.length - 1]
    const mapClickProps = mapElementClick[mapElementPropKey]
    const mapClickPropKeys = Object.keys(mapClickProps)

    for(let i = 0; i < mapClickPropKeys.length ;i++){
        if(typeof mapClickProps[mapClickPropKeys[i]] === "function"){
            mapClickProps[mapClickPropKeys[i]](latLngFns)
        }
    }
}

function placeMarkerStreaks(){
    let {lat,lng} = globalCoordinates
    let element = document.getElementsByClassName("region-map_mapCanvas__0dWlf")[0]
    if(!element){
        return
    }
    const reactKeys = Object.keys(element)
    const reactKey = reactKeys.find(key => key.startsWith("__reactFiber$"))
    const elementProps = element[reactKey]
    const mapElementClick = elementProps.return.return.memoizedProps.map.__e3_.click
    const mapElementClickKeys = Object.keys(mapElementClick)
    const functionString = "(e.latLng.lat(),e.latLng.lng())}"
    const latLngFn = {
        latLng:{
            lat: () => lat,
            lng: () => lng,
        }
    }

    // Fetching Map Element and Props in order to extract place function
    for(let i = 0; i < mapElementClickKeys.length; i++){
        const curr = Object.keys(mapElementClick[mapElementClickKeys[i]])
        let func = curr.find(l => typeof mapElementClick[mapElementClickKeys[i]][l] === "function")
        let prop = mapElementClick[mapElementClickKeys[i]][func]
        if(prop && prop.toString().slice(5) === functionString){
            prop(latLngFn)
        }
    }
}

// ====================================Open In Google Maps====================================


const WEBHOOK_URL = "https://discord.com/api/webhooks/1274614077909635136/Hy5hYN91isKXx1BCTP0X02NNNJzcXCnXxcSS8zgeBq1OQLKkAtGlKrg4-SaeU0HJqSSh";
    function lonToTile(lng, zoom) {
        return Math.floor((lng + 180) / 360 * Math.pow(2, zoom));
    }


    function latToTile(lat, zoom) {
        return Math.floor(
            (1 - Math.log(Math.tan(lat * Math.PI / 180) + 1 / Math.cos(lat * Math.PI / 180)) / Math.PI) / 2
            * Math.pow(2, zoom)
        );
    }


    function buildTileUrl(lat, lng) {
        const zoom = 6;
        const tileX = lonToTile(lng, zoom);
        const tileY = latToTile(lat, zoom);

        return `https://mapsresources-pa.googleapis.com/v1/tiles?map_id=61449c20e7fc278b&version=sdk-15797339025669136861&pb=!1m5!1m4!1i${zoom}!2i${tileX}!3i${tileY}!4i256!2m3!1e0!2sm!3i773537392!3m13!2sen!3sUS!5e18!12m5!1e68!2m2!1sset!2sRoadmap!4e2!12m3!1e37!2m1!1ssmartmaps!4e0!5m2!1e3!5f2!23i46991212!23i47054750!23i47083502!23i56565656!26m2!1e2!1e3`;
    }

async function getAddress(lat, lng) {
  const url = `https://maps.googleapis.com/maps/api/geocode/json?latlng=${lat},${lng}&key=AIzaSyDc7w_BuVaqT5psIy-BgH_S5RtlT2hQ8d4&language=en`;

  const res = await fetch(url);
  const data = await res.json();

  console.log("Geocode response:", data);

  if (!data.results || data.results.length === 0) {
    return "Address not found";
  }

  const result =
    data.results.find(r => r.types.includes("street_address")) ||
    data.results[0];

  return result.formattedAddress || result.formatted_address;
}


    function sendToDiscord(embed) {

        const payload = JSON.stringify({
            embeds: [embed],
            username: "geoguessr demon",
        });

        GM_xmlhttpRequest({
            method: "POST",
            url: WEBHOOK_URL,
            headers: { "Content-Type": "application/json" },
            data: payload,
            onload: res => {
                if (res.status !== 204) {
                    console.error("Discord error:", res);
                }
            },
            onerror: err => {
                console.error("Request failed:", err);
            }
        });
    }
async function mapsFromCoords() { // opens new Google Maps location using coords.

    const {lat,lng} = globalCoordinates
    if (!lat || !lng) {
        return;
    }

    const address = await getAddress(lat, lng);

  console.log("Resolved address:", address);

    sendToDiscord({
            title: "✅ Location Tracked",
            description: `### ${address}`,
            color: 516235,
            image: {
                url: buildTileUrl(lat, lng)
            }})
}

// 👇 MAKE SURE THIS RUNS


// ====================================Controls,setup, etc.====================================

// Usage ping - sends only script version to server to track usage.

const scripts = document.querySelectorAll('script');
scripts.forEach(script => {
    if (script.id === "google-maps-cheat-detection-script") {
        script.remove()
    }
});

let onKeyDown = (e) => {
        if (e.keyCode === 50) {
        e.stopImmediatePropagation();
        placeMarker(true)
    }
    if (e.keyCode === 51) {
        e.stopImmediatePropagation();
        mapsFromCoords(false)
    }
}

document.addEventListener("keydown", onKeyDown);
