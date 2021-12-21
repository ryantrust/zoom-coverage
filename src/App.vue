<template>
    <div id="root">
        <div id="map"></div>
        <div class="overlay top left flex-col gap">

            <Button
            @click="clearMarkers"
            class="bg-warning"
            text="Clear markers"
            optText="(for performance, this won't erase your generated locations)"
            title="Clear markers"
            />
        </div>

        <div class="overlay top right flex-col gap">
            <div class="settings">
                <input id="fromX" v-model.number="userFromX" placeholder="X1" type="number"/>,<input id="fromY" v-model.number="userFromY" placeholder="Y1" type="number"/> to <input id="toX" v-model.number="userToX" placeholder="X2" type="number"/>, <input id="toY" v-model.number="userToY" placeholder="Y2" type="number"/>

                <Button @click="getUserTileRange()" class="bordered-success" text="Check Tile Range" title="Check Tile Range" />
                <hr/>

                <div class="space-between">
                    <Checkbox v-model:checked="settings.cluster" v-on:change="checkClusters" label="Cluster markers" />
                    <small>For lag reduction.</small>
                </div>

                <hr/>
                <div class="space-between">
                    <Checkbox v-model:checked="settings.checkLinked" label="Check linked panos" />
                    <small>Only works for coverage accessable by API.</small>
                </div>

            </div>
        </div>

        <div class="overlay export bottom right">
            <h4 class="center mb-2">Export locations to</h4>
            <div class="flex gap">
                <Button @click="exportToJSON()" class="bordered-success" text="JSON" title="Export as JSON" />
                <Button @click="exportToCSV()" class="bordered-success" text="CSV" title="Export as CSV"/>
            </div>
        </div>
    </div>
</template>


<script setup>
import { onMounted, ref, reactive, computed } from "vue";
import Button from "./components/Button.vue";
import Checkbox from "./components/Checkbox.vue";

import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet-draw/dist/leaflet.draw.js";
import "leaflet-draw/dist/leaflet.draw.css";

import "leaflet.markercluster/dist/leaflet.markercluster.js";
import "leaflet.markercluster/dist/MarkerCluster.css";
import "leaflet.markercluster/dist/MarkerCluster.Default.css";

import "leaflet.markercluster.freezable/dist/leaflet.markercluster.freezable.js";

import "leaflet-contextmenu/dist/leaflet.contextmenu.js";
import "leaflet-contextmenu/dist/leaflet.contextmenu.css";

import booleanPointInPolygon from "@turf/boolean-point-in-polygon";

import axios from "axios";

import PromisePool from "@supercharge/promise-pool";

let userFromX = null;
let userFromY = null;
let userToX = null;
let userToY = null;

const settings = reactive({
    cluster: false,
    checkLinked: false
});

const defaultMarker = "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABkAAAApCAYAAADAk4LOAAAFgUlEQVR4Aa1XA5BjWRTN2oW17d3YaZtr2962HUzbDNpjszW24mRt28p47v7zq/bXZtrp/lWnXr337j3nPCe85NcypgSFdugCpW5YoDAMRaIMqRi6aKq5E3YqDQO3qAwjVWrD8Ncq/RBpykd8oZUb/kaJutow8r1aP9II0WmLKLIsJyv1w/kqw9Ch2MYdB++12Onxee/QMwvf4/Dk/Lfp/i4nxTXtOoQ4pW5Aj7wpici1A9erdAN2OH64x8OSP9j3Ft3b7aWkTg/Fm91siTra0f9on5sQr9INejH6CUUUpavjFNq1B+Oadhxmnfa8RfEmN8VNAsQhPqF55xHkMzz3jSmChWU6f7/XZKNH+9+hBLOHYozuKQPxyMPUKkrX/K0uWnfFaJGS1QPRtZsOPtr3NsW0uyh6NNCOkU3Yz+bXbT3I8G3xE5EXLXtCXbbqwCO9zPQYPRTZ5vIDXD7U+w7rFDEoUUf7ibHIR4y6bLVPXrz8JVZEql13trxwue/uDivd3fkWRbS6/IA2bID4uk0UpF1N8qLlbBlXs4Ee7HLTfV1j54APvODnSfOWBqtKVvjgLKzF5YdEk5ewRkGlK0i33Eofffc7HT56jD7/6U+qH3Cx7SBLNntH5YIPvODnyfIXZYRVDPqgHtLs5ABHD3YzLuespb7t79FY34DjMwrVrcTuwlT55YMPvOBnRrJ4VXTdNnYug5ucHLBjEpt30701A3Ts+HEa73u6dT3FNWwflY86eMHPk+Yu+i6pzUpRrW7SNDg5JHR4KapmM5Wv2E8Tfcb1HoqqHMHU+uWDD7zg54mz5/2BSnizi9T1Dg4QQXLToGNCkb6tb1NU+QAlGr1++eADrzhn/u8Q2YZhQVlZ5+CAOtqfbhmaUCS1ezNFVm2imDbPmPng5wmz+gwh+oHDce0eUtQ6OGDIyR0uUhUsoO3vfDmmgOezH0mZN59x7MBi++WDL1g/eEiU3avlidO671bkLfwbw5XV2P8Pzo0ydy4t2/0eu33xYSOMOD8hTf4CrBtGMSoXfPLchX+J0ruSePw3LZeK0juPJbYzrhkH0io7B3k164hiGvawhOKMLkrQLyVpZg8rHFW7E2uHOL888IBPlNZ1FPzstSJM694fWr6RwpvcJK60+0HCILTBzZLFNdtAzJaohze60T8qBzyh5ZuOg5e7uwQppofEmf2++DYvmySqGBuKaicF1blQjhuHdvCIMvp8whTTfZzI7RldpwtSzL+F1+wkdZ2TBOW2gIF88PBTzD/gpeREAMEbxnJcaJHNHrpzji0gQCS6hdkEeYt9DF/2qPcEC8RM28Hwmr3sdNyht00byAut2k3gufWNtgtOEOFGUwcXWNDbdNbpgBGxEvKkOQsxivJx33iow0Vw5S6SVTrpVq11ysA2Rp7gTfPfktc6zhtXBBC+adRLshf6sG2RfHPZ5EAc4sVZ83yCN00Fk/4kggu40ZTvIEm5g24qtU4KjBrx/BTTH8ifVASAG7gKrnWxJDcU7x8X6Ecczhm3o6YicvsLXWfh3Ch1W0k8x0nXF+0fFxgt4phz8QvypiwCCFKMqXCnqXExjq10beH+UUA7+nG6mdG/Pu0f3LgFcGrl2s0kNNjpmoJ9o4B29CMO8dMT4Q5ox8uitF6fqsrJOr8qnwNbRzv6hSnG5wP+64C7h9lp30hKNtKdWjtdkbuPA19nJ7Tz3zR/ibgARbhb4AlhavcBebmTHcFl2fvYEnW0ox9xMxKBS8btJ+KiEbq9zA4RthQXDhPa0T9TEe69gWupwc6uBUphquXgf+/FrIjweHQS4/pduMe5ERUMHUd9xv8ZR98CxkS4F2n3EUrUZ10EYNw7BWm9x1GiPssi3GgiGRDKWRYZfXlON+dfNbM+GgIwYdwAAAAASUVORK5CYII=";

const defaultMarkerShadow = "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8/5+hHgAHggJ/PchI7wAAAABJRU5ErkJggg==";

let defaultIcon = new L.Icon({
    iconUrl: defaultMarker,
    iconAnchor: [12, 41],
    shadowUrl: defaultMarkerShadow,
    shadowSize: [0,0]
});

const checkedPanos = new Set();
const checkedTiles = new Set();
let panos = {};
let map;
const SV = new google.maps.StreetViewService();

const markerLayer = L.markerClusterGroup({
    maxClusterRadius: 25,
    disableClusteringAtZoom: 20
});

markerLayer.disableClustering();

const customPolygonsLayer = new L.FeatureGroup();
const drawControl = new L.Control.Draw({
    position: "bottomleft",
    draw: {
        polyline: false,
        marker: false,
        circlemarker: false,
        circle: false,
        polygon: {
            allowIntersection: false,
            drawError: {
                color: "#e1e100",
                message: "<strong>Polygon draw does not allow intersections!<strong> (allowIntersection: false)"
            },
            shapeOptions: { color: "#5d8ce3" }
        },
        rectangle: { shapeOptions: { color: "#5d8ce3" } }
    },
    edit: { featureGroup: customPolygonsLayer }
});

L.GridLayer.DebugCoords = L.GridLayer.extend({
    createTile: function (coords) {
        const tile = document.createElement("div");
        if(coords.z==17&&checkedTiles.has(coords.x+","+coords.y))tile.style.outline = "1px solid green";
        else tile.style.outline = "1px solid red";
        tile.style.color = "black";
        tile.style.fontWeight = "bold";
        tile.style.fontSize = "14pt";
        tile.innerHTML = [coords.z, coords.x, coords.y].join("/");
        return tile;
    }
});

L.gridLayer.debugCoords = function(opts) {
    return new L.GridLayer.DebugCoords(opts);
};

let debugLayer = L.gridLayer.debugCoords({minZoom:16,maxZoom:20,maxNativeZoom: 17,minNativeZoom: 17});

function checkClusters() {
    if(settings.cluster) markerLayer.enableClustering();
    else markerLayer.disableClustering();
}

function coordsInBounds(coords) {
    if(!Object.keys(customPolygonsLayer._layers).length) return true;
    for (var bounds of Object.values(customPolygonsLayer._layers)) {
        if(!booleanPointInPolygon([coords[1], coords[0]], bounds.toGeoJSON())) return false;
    }
    return true;
}

function checkCoords(lat, lng) {
    SV.getPanorama(
        {
            location: { lat: lat, lng: lng },
            preference: google.maps.StreetViewPreference.NEAREST
        },
        (res, status) => {
            if (status === "ZERO_RESULTS") return;
            if (status !== "OK") {
                console.error("Error at " + lat + " , " + lng + " : " + status);
                return;
            } // no result
            checkPano(res.location.pano);

            for (var g of res.time) checkPano(g.pano);
            for (var h of res.links) checkPano(h.pano);
        }
    );
}

function checkPano(id) {
    if (checkedPanos.has(id)) return;
    else checkedPanos.add(id);

    SV.getPanorama({ pano: id }, (res, status) => {
        if (status === "UNKNOWN_ERROR") {
            console.warn("Unknown error with " + id + " , trying again.");
            checkedPanos.delete(id);
            checkPano(id);
            return;
        }
        if (status !== "OK") {
            // no result?
            console.error("Uncaught error with Pano ID result?: " + id + " , " + status);
            return;
        }
        if(!coordsInBounds([res.location.latLng.lat(),res.location.latLng.lng()])) {
            checkedPanos.delete(id);
            return;
        }

        if(!(res.location.pano in panos)) {
            panos[res.location.pano] = {
                lat: res.location.latLng.lat(),
                lng: res.location.latLng.lng(),
                date: res.imageDate
            };

            L.marker([res.location.latLng.lat(), res.location.latLng.lng()], {
                icon: defaultIcon,
                title: id,
                opacity: 0.75
            })
            .on("click", () => {
                window.open(
                    `https://www.google.com/maps/@?api=1&map_action=pano&pano=${
                        res.location.pano
                    }
                    ${location.heading ? "&heading=" + location.heading : ""}
                    ${location.pitch ? "&pitch=" + location.pitch : ""}`,
                    "_blank"
                );
            })
            .addTo(markerLayer);
        }

        checkCoords(res.location.latLng.lat(), res.location.latLng.lng());
        for (var g of res.time) checkPano(g.pano);
        for (var h of res.links) checkPano(h.pano);


    });
}

function exportToJSON() {
    const dataUri = "data:application/json;charset=utf-8," + encodeURIComponent(JSON.stringify(panos));
    const fileName = `Generated map.json`;
    const linkElement = document.createElement("a");
    linkElement.href = dataUri;
    linkElement.download = fileName;
    linkElement.click();
}

function exportToCSV() {
    let csv = "";
    for(var h of Object.values(panos)) {
        csv+=h.lat+","+h.lng+"\n";
    }
    const dataUri = "data:application/json;charset=utf-8," + encodeURIComponent(csv);
    const fileName = `Generated map.csv`;
    const linkElement = document.createElement("a");
    linkElement.href = dataUri;
    linkElement.download = fileName;
    linkElement.click();
}

function customPolygonStyle() {
    return {
        weight: 2,
        opacity: 1,
        color: getRandomColor(),
        fillOpacity: 0,
    };
}

function getRandomColor() {
    const red = Math.floor(((1 + Math.random()) * 256) / 2);
    const green = Math.floor(((1 + Math.random()) * 256) / 2);
    const blue = Math.floor(((1 + Math.random()) * 256) / 2);
    return "rgb(" + red + ", " + green + ", " + blue + ")";
}

function clearMarkers() {
    markerLayer.clearLayers();
}

function project(lat, lng) {
    let siny = Math.sin((lat * Math.PI) / 180);

    // Truncating to 0.9999 effectively limits latitude to 89.189. This is
    // about a third of a tile past the edge of the world tile.
    siny = Math.min(Math.max(siny, -0.9999), 0.9999);
    return [256 * (0.5 + lng / 360), 256 * (0.5 - Math.log((1 + siny) / (1 - siny)) / (4 * Math.PI))];
}

function coordsToTile(lat, lng, zoom) {
    const worldCoords = project(lat, lng);
    const scale = 1 << zoom;
    return [Math.floor((worldCoords[0] * scale) / 256), Math.floor((worldCoords[1] * scale) / 256)];
}

function range(fromX, toX, fromY, toY) {
    let values = [];
    let lessX = Math.min(fromX, toX);
    let moreX = Math.max(fromX, toX);
    let lessY = Math.min(fromY, toY);
    let moreY = Math.max(fromY, toY);
    for(let x=lessX;x<=moreX;x++) {
        for(let y=lessY;y<=moreY;y++) {
            values.push([x,y]);
        }
    }
    return values;
}

function onClick(e) {
    const tile = coordsToTile(e.latlng.lat, e.latlng.lng, 17);
    getTile(tile[0], tile[1]);
}

function processZoomData(data) {
    let d = JSON.parse("["+data.substr(data.indexOf("\n")+2))[1];
    if(!d || !d[1]) return [];
    let list = d[1];
    let ret = [];
    for(let item of Object.values(list)) {
        ret.push({pano:item[0][0][1],lat:item[0][2][0][2],lng:item[0][2][0][3]});
    }
    return ret;
}

function getTile(x, y) {
    if(checkedTiles.has(x+","+y))return;
    checkedTiles.add(x+","+y);
    return axios.get("https://www.google.com/maps/photometa/ac/v1?pb=!1m1!1smaps_sv.tactile!6m3!1i"+x+"!2i"+y+"!3i17!8b1")
    .then(function (response) {
        let data = processZoomData(response.data);
        if(!data) return;
        for(let pano of data) {
            if(pano.pano in panos) continue;
            panos[pano.pano] = {
                lat: pano.lat,
                lng: pano.lng,
            };
            L.marker([pano.lat, pano.lng], {
                icon: defaultIcon,
                title: pano.pano,
                opacity: 0.75
            })
            .on("click", () => {
                window.open(
                    `https://www.google.com/maps/@?api=1&map_action=pano&pano=${
                        pano.pano
                    }
                    ${location.heading ? "&heading=" + location.heading : ""}
                    ${location.pitch ? "&pitch=" + location.pitch : ""}`,
                    "_blank"
                );
            })
            .addTo(markerLayer);
            if(settings.checkLinked)checkPano(pano.pano);
        }
    })
    .catch(function (error) {
        console.error("Error getting tile "+x+", "+y+", trying again.")
        checkedTiles.delete(x+","+y);
        getTile(x,y);
    });
}

async function getTileRange(fromX, toX, fromY, toY) {
    const { results, errors } = await PromisePool
    .for(range(fromX, toX, fromY, toY))
    .withConcurrency(100)
    .process(async data => {
        return await getTile(data[0], data[1]);
    });
    debugLayer.redraw();
    return {results:results,errors:errors};
    //TODO MAKE SINGLE PROMISEPOOL FOR WHOLE PROGRAM
}

function getUserTileRange() {
    return getTileRange(userFromX, userToX, userFromY, userToY);
}

function setCorner1(e) {
    let tile = coordsToTile(e.latlng.lat, e.latlng.lng, 17);
    userFromX = tile[0];
    userFromY = tile[1];
    document.getElementById("fromX").value = tile[0];
    document.getElementById("fromY").value = tile[1];
}

function setCorner2(e) {
    let tile = coordsToTile(e.latlng.lat, e.latlng.lng, 17);
    userToX = tile[0];
    userToY = tile[1];
    document.getElementById("toX").value = tile[0];
    document.getElementById("toY").value = tile[1];
}

onMounted(function() {
    map = L.map("map", {
        attributionControl: false,
        contextmenu: true,
        contextmenuItems: [
            {text:"Set Corner 1",callback:setCorner1},
            {text:"Set Corner 2",callback:setCorner2}
        ],
        center: [48.15619,7.61215],
        preferCanvas: true,
        zoom: 16,
        zoomControl: false,
        worldCopyJump: true,
        layers: [
            L.tileLayer("https://{s}.google.com/vt/lyrs=m&hl=en&x={x}&y={y}&z={z}", {
                subdomains: ["mt0", "mt1", "mt2", "mt3"],
                type: "roadmap",
                maxZoom: 26,
                maxNativeZoom: 22
            })
        ]
    });


    customPolygonsLayer.addTo(map);
    map.addControl(drawControl);

    map.addLayer(debugLayer);

    map.on("draw:created", (e) => {
        const polygon = e.layer;
        polygon.feature = e.layer.toGeoJSON();
        polygon.found = [];
        polygon.nbNeeded = 100;
        polygon.setStyle(customPolygonStyle());
        customPolygonsLayer.addLayer(polygon);
    });


    map.on("click", onClick);

    markerLayer.addTo(map);

    checkClusters();

    //getTileRange(68932, 68937, 41581, 41600);
});


</script>

<style>
@import "./assets/main.css";
#map {
    z-index: 0;
    height: 100vh;
}
.leaflet-container {
    background-color: #2c2c2c;
}
.overlay {
    position: absolute;
}
.select,
.selected,
.settings,
.export {
    padding: var(--space-2);
    border-radius: 5px;
    background: rgba(0, 0, 0, 0.7);
    box-shadow: 0 0 2px rgba(0, 0, 0, 0.4);
}
.selected {
    max-height: calc(100vh - 390px);
    overflow: auto;
}
.settings {
    max-width: 375px;
    max-height: calc(100vh - 180px);
    overflow: auto;
}
.export {
    min-width: 375px;
}
.line {
    line-height: 1.5rem;
}
</style>
