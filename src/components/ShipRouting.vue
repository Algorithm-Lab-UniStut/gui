<template>
    <v-container>
        <v-row justify="end">
            <v-col class="pt-0">
                <div class="d-flex align-end flex-column">
                    <v-switch
                        v-model="leafletSwitch"
                        label="Leaflet"
                        hide-details
                        class="float-right"
                    ></v-switch>
                    <v-switch
                        v-model="cesiumSwitch"
                        label="Cesium"
                        hide-details
                        class="float-right"
                    ></v-switch>
                </div>
            </v-col>
        </v-row>
        <v-row justify="center">
            <div id="leafletMap" class="mb-2" v-if="leafletSwitch"></div>
        </v-row>
        <v-row class="mb-1">
            <div id="cesiumContainer" v-if="cesiumSwitch"></div>
        </v-row>
        <v-row class="py-1 mt-1" align="center">
            <v-col cols="2">
                <v-icon color="blue darken-2" small>mdi-map-marker</v-icon
                >Origin:
            </v-col>
            <v-col cols="2">
                <v-text-field
                    label="Latitude"
                    v-model="originInput.lat"
                    suffix="°"
                    readonly
                ></v-text-field
            ></v-col>
            <v-col cols="2">
                <v-text-field
                    label="Longitude"
                    v-model="originInput.lng"
                    suffix="°"
                    readonly
                ></v-text-field>
            </v-col>
            <v-col cols="2"
                ><v-icon color="red darken-2" small>mdi-map-marker</v-icon
                >Destination:</v-col
            >
            <v-col cols="2">
                <v-text-field
                    label="Latitude"
                    v-model="destinationInput.lat"
                    suffix="°"
                    readonly
                ></v-text-field
            ></v-col>
            <v-col cols="2">
                <v-text-field
                    label="Longitude"
                    v-model="destinationInput.lng"
                    suffix="°"
                    readonly
                ></v-text-field>
            </v-col>
        </v-row>
        <v-row class="mb-1">
            <v-btn
                color="primary"
                block
                id="navigateBtn"
                :disabled="navigating || !origin || !destination"
                :loading="navigating"
                @click="computeRoute"
            >
                Navigate
            </v-btn>
            <v-banner class="pa-2 my-4" v-if="path.waypoints && !navigating">
                {{
                    path.reachable
                        ? "Length: " + path.length / 1000 + "km"
                        : "Not reachable"
                }}
            </v-banner>
        </v-row>
        <v-row class="mb-1">
            <v-btn
                color="primary"
                block
                id="searchSpaceBtn"
                :disabled="graph.loadSearchSpace"
                :loading="graph.loadSearchSpace"
                @click="toggleSearchSpace"
            >
                {{
                    graph.showSearchSpace
                        ? "Hide Search Space"
                        : "Show Search Space"
                }}
            </v-btn>
        </v-row>
        <v-row class="mb-1">
            <v-btn
                color="primary"
                block
                id="loadGraphNodesBtn"
                :disabled="graph.loadVertices"
                :loading="graph.loadVertices"
                @click="toggleGraphNodes"
            >
                {{
                    graph.showVertices ? "Hide Graph Nodes" : "Show Graph Nodes"
                }}
            </v-btn>
        </v-row>
    </v-container>
</template>

<script>
/* global Cesium, L */
/* es-lint no-undef: "error" */
import axios from "axios";

export default {
    name: "ShipRouting",
    data: () => ({
        path: { waypoints: null, length: 0, reachable: true },
        navigating: false,
        graph: {
            vertices: null,
            showVertices: false,
            loadVertices: false,
            searchSpace: null,
            loadSearchSpace: false,
            showSearchSpace: false,
        },
        cesiumSwitch: true,
        leafletSwitch: false,
        originInput: { lat: null, lng: null },
        destinationInput: { lat: null, lng: null },
        leaflet: {
            map: null,
            originMarker: L.marker(),
            destinationMarker: L.marker(),
            directPath: null,
            path: null,
            graphVerticeMarker: null,
            searchSpaceVerticeMarker: null,
            verticeRenderer: L.canvas({ padding: 0.5 }),
        },
        cesium: {
            viewer: null,
            originMarker: null,
            destinationMarker: null,
            greatCircleLine: null,
            rhumbLine: null,
            path: null,
            graphVerticeMarker: null,
            searchSpaceVerticeMarker: null,
        },
    }),
    computed: {
        origin() {
            if (this.cesiumSwitch) {
                if (!this.cesium.originMarker) return null;
                const cartographic = Cesium.Cartographic.fromCartesian(
                    this.cesium.originMarker._position._value
                );
                return {
                    lat: Cesium.Math.toDegrees(cartographic.latitude),
                    lng: Cesium.Math.toDegrees(cartographic.longitude),
                };
            } else if (this.leafletSwitch) {
                return this.leaflet.originMarker._latlng;
            }
            return null;
        },
        destination() {
            if (this.cesiumSwitch) {
                if (!this.cesium.destinationMarker) return null;
                const cartographic = Cesium.Cartographic.fromCartesian(
                    this.cesium.destinationMarker._position._value
                );
                return {
                    lat: Cesium.Math.toDegrees(cartographic.latitude),
                    lng: Cesium.Math.toDegrees(cartographic.longitude),
                };
            } else if (this.leafletSwitch) {
                return this.leaflet.destinationMarker._latlng;
            }
            return null;
        },
    },
    watch: {
        async leafletSwitch() {
            this.cesiumSwitch = !this.leafletSwitch;
            if (this.leafletSwitch) {
                await this.$nextTick();
                this.setupLeaflet();
                if (this.cesium.originMarker) {
                    const cartographic = Cesium.Cartographic.fromCartesian(
                        this.cesium.originMarker._position._value
                    );
                    const lat = Cesium.Math.toDegrees(cartographic.latitude);
                    const lng = Cesium.Math.toDegrees(cartographic.longitude);
                    this.leafletPlaceOriginMarker({ lat, lng });
                }
                if (this.cesium.destinationMarker) {
                    const cartographic = Cesium.Cartographic.fromCartesian(
                        this.cesium.destinationMarker._position._value
                    );
                    const lat = Cesium.Math.toDegrees(cartographic.latitude);
                    const lng = Cesium.Math.toDegrees(cartographic.longitude);
                    this.leafletPlaceDestinationMarker({ lat, lng });
                }

                const directPath =
                    this.origin && this.destination
                        ? [this.origin, this.destination]
                        : [];
                this.leaflet.directPath = this.drawLeafletLine(
                    this.leaflet.directPath,
                    directPath
                );
                if (this.path.waypoints) {
                    this.leaflet.path = this.drawLeafletLine(
                        this.leaflet.path,
                        this.path.waypoints.map((w) => {
                            return { lat: w.lat, lng: w.lon };
                        }),
                        "white"
                    );
                }
            } else {
                this.teardownLeaflet();
            }
        },
        async cesiumSwitch() {
            this.leafletSwitch = !this.cesiumSwitch;
            if (this.cesiumSwitch) {
                await this.$nextTick();
                this.setupCesium();
                if (this.leaflet.originMarker._latlng) {
                    this.cesiumPlaceOriginMarker(
                        this.leaflet.originMarker._latlng
                    );
                }

                if (this.leaflet.destinationMarker._latlng) {
                    this.cesiumPlaceDestinationMarker(
                        this.leaflet.destinationMarker._latlng
                    );
                }

                const waypointsDegreesArray = this.path.waypoints
                    ? this.path.waypoints.map((w) => [w.lon, w.lat]).flat()
                    : [];
                const directPath =
                    this.origin && this.destination
                        ? [this.origin, this.destination]
                        : [];
                const directPathDegreesArray = directPath
                    .map((w) => [w.lng, w.lat])
                    .flat();
                this.cesium.path = this.drawCesiumLine(
                    this.cesium.path,
                    waypointsDegreesArray,
                    Cesium.ArcType.GEODESIC,
                    "Ship route",
                    Cesium.Color.WHITE,
                    3
                );
                this.cesium.rhumbLine = this.drawCesiumLine(
                    this.cesium.rhumbLine,
                    directPathDegreesArray,
                    Cesium.ArcType.RHUMB,
                    "Green rhumb line on terrain",
                    Cesium.Color.GREEN,
                    3
                );
                this.cesium.greatCircleLine = this.drawCesiumLine(
                    this.cesium.greatCircleLine,
                    directPathDegreesArray,
                    Cesium.ArcType.GEODESIC,
                    "Red line on terrain",
                    Cesium.Color.RED,
                    3
                );
            } else {
                this.teardownCesium();
            }
        },
        origin() {
            if (this.origin) {
                if (this.origin.lat) this.originInput.lat = this.origin.lat;
                if (this.origin.lng) this.originInput.lng = this.origin.lng;
            } else {
                this.originInput.lat = null;
                this.originInput.lng = null;
            }
        },
        destination() {
            if (this.destination) {
                if (this.destination.lat)
                    this.destinationInput.lat = this.destination.lat;
                if (this.destination.lng)
                    this.destinationInput.lng = this.destination.lng;
            } else {
                this.destinationInput.lat = null;
                this.destinationInput.lng = null;
            }
        },
    },
    mounted() {
        if (this.cesiumSwitch) this.setupCesium();
        if (this.leafletSwitch) this.setupLeaflet();
    },
    methods: {
        async computeRoute() {
            this.navigating = true;
            this.path.waypoints = [];
            this.path.length = 0;
            try {
                const backend = process.env.VUE_APP_BACKEND_URL;
                const port = process.env.VUE_APP_BACKEND_PORT;
                const { data } = await axios.post(
                    `${backend}:${port}/routes`,
                    JSON.stringify({
                        origin: { lon: this.origin.lng, lat: this.origin.lat },
                        destination: {
                            lon: this.destination.lng,
                            lat: this.destination.lat,
                        },
                    })
                );
                console.log(data);
                this.path.reachable = data.reachable;
                if (data.reachable) {
                    this.path.waypoints = data.path.waypoints;
                    this.path.length = data.path.length;
                }
            } catch (e) {
                this.path.reachable = false;
                console.error(e);
            } finally {
                const waypointsDegreesArray = this.path.waypoints
                    .map((w) => [w.lon, w.lat])
                    .flat();
                const directPath = [this.origin, this.destination];
                const directPathDegreesArray = directPath
                    .map((w) => [w.lng, w.lat])
                    .flat();
                if (this.cesiumSwitch) {
                    this.cesium.path = this.drawCesiumLine(
                        this.cesium.path,
                        waypointsDegreesArray,
                        Cesium.ArcType.GEODESIC,
                        "Ship route",
                        Cesium.Color.WHITE,
                        3
                    );
                    this.cesium.rhumbLine = this.drawCesiumLine(
                        this.cesium.rhumbLine,
                        directPathDegreesArray,
                        Cesium.ArcType.RHUMB,
                        "Green rhumb line on terrain",
                        Cesium.Color.GREEN,
                        3
                    );
                    this.cesium.greatCircleLine = this.drawCesiumLine(
                        this.cesium.greatCircleLine,
                        directPathDegreesArray,
                        Cesium.ArcType.GEODESIC,
                        "Red line on terrain",
                        Cesium.Color.RED,
                        3
                    );
                }
                if (this.leafletSwitch) {
                    this.leaflet.directPath = this.drawLeafletLine(
                        this.leaflet.directPath,
                        directPath
                    );
                    this.leaflet.path = this.drawLeafletLine(
                        this.leaflet.path,
                        this.path.waypoints.map((w) => {
                            return { lat: w.lat, lng: w.lon };
                        }),
                        "white"
                    );
                }
                this.navigating = false;
            }
        },
        async toggleGraphNodes() {
            this.graph.loadVertices = true;
            await this.getGraphNodes();
            if (this.leafletSwitch) {
                if (!this.graph.showVertices) {
                    this.leafletShowGraphNodes(this.graph.vertices);
                    this.graph.showVertices = true;
                } else {
                    this.leafletHideGraphNodes();
                    this.graph.showVertices = false;
                }
            }
            if (this.cesiumSwitch) {
                if (!this.graph.showVertices) {
                    this.cesiumShowGraphNodes(this.graph.vertices);
                    this.graph.showVertices = true;
                } else {
                    this.cesiumHideGraphNodes();
                    this.graph.showVertices = false;
                }
            }
            this.graph.loadVertices = false;
        },
        async toggleSearchSpace() {
            this.graph.loadSearchSpace = true;
            await this.getSearchSpace();
            if (this.leafletSwitch) {
                if (!this.graph.showSearchSpace) {
                    this.leafletShowSearchSpace(this.graph.searchSpace);
                    this.graph.showSearchSpace = true;
                } else {
                    this.leafletHideSearchSpace();
                    this.graph.showSearchSpace = false;
                }
            }
            if (this.cesiumSwitch) {
                if (!this.graph.showSearchSpace) {
                    this.cesiumShowSearchSpace(this.graph.searchSpace);
                    this.graph.showSearchSpace = true;
                } else {
                    this.cesiumHideSearchSpace();
                    this.graph.showSearchSpace = false;
                }
            }
            this.graph.loadSearchSpace = false;
        },
        async getGraphNodes() {
            if (this.graph.vertices == null) {
                try {
                    const { data } = await axios.get(
                        "http://localhost:8081/nodes"
                    );
                    this.graph.vertices = data.waypoints;
                    //console.log(data.waypoints);
                } catch (e) {
                    console.error(e);
                }
            }
        },
        async getSearchSpace() {
            try {
                const { data } = await axios.get(
                    "http://localhost:8081/searchSpace"
                );
                this.graph.searchSpace = data.waypoints;
            } catch (e) {
                console.error(e);
                this.graph.searchSpace = null;
            }
        },
        setupCesium() {
            Cesium.Ion.defaultAccessToken =
                "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiIxMDk0MThjNS0yODUyLTRiYTItOTc0MC0wNjhjZmJkNWFmYjQiLCJpZCI6OTQ0NjYsImlhdCI6MTY1Mjk2NTE2Mn0.fyZexWFaRBx0YQqQEJnxL-BtmObI3uXfxcPvNEi7LFg";

            const extent = Cesium.Rectangle.fromDegrees(-1, 19, 3, 17);

            Cesium.Camera.DEFAULT_VIEW_RECTANGLE = extent;
            Cesium.Camera.DEFAULT_VIEW_FACTOR = 3;

            // Initialize the Cesium Viewer in the HTML element with the "cesiumContainer" ID.
            let imageryProvider = new Cesium.ArcGisMapServerImageryProvider({
                url: "https://services.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer",
            });
            imageryProvider = Cesium.createWorldImagery({
                style: Cesium.IonWorldImageryStyle.AERIAL_WITH_LABELS,
            });
            this.cesium.viewer = new Cesium.Viewer("cesiumContainer", {
                //terrainProvider: Cesium.createWorldTerrain(),
                imageryProvider,
                selectionIndicator: false,
                infoBox: false,
                animation: false,
                timeline: false,
                fullscreenButton: false,
                baseLayerPicker: false, // select different layers (satellite, night, OSM, world terrain), but some are not working
                //sceneModePicker: false,
                //navigationHelpButton: false,
                //homeButton: false,
                //geocoder: false,
            });

            const scene = this.cesium.viewer.scene;
            if (!scene.pickPositionSupported) {
                window.alert("This browser does not support pickPosition.");
            }
            // Mouse over the globe to see the cartographic position
            const handler = new Cesium.ScreenSpaceEventHandler(scene.canvas);
            handler.setInputAction((movement) => {
                let position = this.cesium.viewer.scene.camera.pickEllipsoid(
                    movement.position,
                    this.cesium.viewer.scene.globe.ellipsoid
                );
                if (!position) {
                    console.log("Globe was not picked");
                    return;
                }
                let cartographic = Cesium.Cartographic.fromCartesian(position);
                let lat = Cesium.Math.toDegrees(cartographic.latitude);
                let lng = Cesium.Math.toDegrees(cartographic.longitude);
                if (this.cesium.originMarker)
                    this.cesium.viewer.entities.remove(
                        this.cesium.originMarker
                    );
                this.cesium.originMarker = this.cesium.viewer.entities.add({
                    position: Cesium.Cartesian3.fromDegrees(lng, lat),
                    billboard: {
                        image: require("@/assets/blue-marker.png"),
                        //pixelOffset: new Cesium.Cartesian2(0, -50), // default: (0, 0)
                        horizontalOrigin: Cesium.HorizontalOrigin.CENTER, // default
                        verticalOrigin: Cesium.VerticalOrigin.BOTTOM, // default: CENTER
                        scale: 0.3,
                        //disableDepthTestDistance: Number.POSITIVE_INFINITY, // necessary when using Cesium World Terrain because of rendering issues
                    },
                });
            }, Cesium.ScreenSpaceEventType.LEFT_CLICK);
            handler.setInputAction((movement) => {
                let position = this.cesium.viewer.scene.camera.pickEllipsoid(
                    movement.position,
                    this.cesium.viewer.scene.globe.ellipsoid
                );
                if (!position) {
                    console.log("Globe was not picked");
                    return;
                }
                let cartographic = Cesium.Cartographic.fromCartesian(position);
                let lat = Cesium.Math.toDegrees(cartographic.latitude);
                let lng = Cesium.Math.toDegrees(cartographic.longitude);
                if (this.cesium.destinationMarker)
                    this.cesium.viewer.entities.remove(
                        this.cesium.destinationMarker
                    );
                this.cesium.destinationMarker = this.cesium.viewer.entities.add(
                    {
                        position: Cesium.Cartesian3.fromDegrees(lng, lat),
                        billboard: {
                            image: require("@/assets/red-marker.png"),
                            //pixelOffset: new Cesium.Cartesian2(0, -50), // default: (0, 0)
                            horizontalOrigin: Cesium.HorizontalOrigin.CENTER, // default
                            verticalOrigin: Cesium.VerticalOrigin.BOTTOM, // default: CENTER
                            scale: 0.3,
                            //disableDepthTestDistance: Number.POSITIVE_INFINITY, // necessary when using Cesium World Terrain because of rendering issues
                        },
                    }
                );
            }, Cesium.ScreenSpaceEventType.RIGHT_CLICK);
        },
        teardownCesium() {
            this.cesium.viewer.destroy();
        },
        cesiumPlaceMarker(image, latlng) {
            return this.cesium.viewer.entities.add({
                position: Cesium.Cartesian3.fromDegrees(latlng.lng, latlng.lat),
                billboard: {
                    image,
                    horizontalOrigin: Cesium.HorizontalOrigin.CENTER, // default
                    verticalOrigin: Cesium.VerticalOrigin.BOTTOM, // default: CENTER
                    scale: 0.3,
                },
            });
        },
        cesiumPlaceOriginMarker(latlng) {
            this.cesium.originMarker = this.cesiumPlaceMarker(
                require("@/assets/blue-marker.png"),
                latlng
            );
        },
        cesiumPlaceDestinationMarker(latlng) {
            this.cesium.destinationMarker = this.cesiumPlaceMarker(
                require("@/assets/red-marker.png"),
                latlng
            );
        },
        drawCesiumLine(line, waypoints, type, name, color, width) {
            this.cesium.viewer.entities.remove(line);
            const path = this.cesium.viewer.entities.add({
                name: name,
                polyline: {
                    positions: Cesium.Cartesian3.fromDegreesArray(waypoints),
                    width: width,
                    material: color,
                    arcType: type,
                    clampToGround: true,
                },
            });
            return path;
        },
        setupLeaflet() {
            this.leaflet.map = L.map("leafletMap").setView([0, 0], 2);
            const accessToken =
                "pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw";
            L.tileLayer(
                `https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token=${accessToken}`,
                {
                    maxZoom: 18,
                    attribution:
                        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>, ' +
                        '&copy; <a href="https://www.mapbox.com/">Mapbox</a>',
                    id: "mapbox/streets-v11",
                    tileSize: 512,
                    zoomOffset: -1,
                }
            ).addTo(this.leaflet.map);
            this.leaflet.originMarker = L.marker();
            this.leaflet.destinationMarker = L.marker();
            // left click: place origin marker
            this.leaflet.map.on("click", (e) =>
                this.leafletPlaceOriginMarker(e.latlng)
            );
            // abuse the contextmenu (right click) for placing destination
            this.leaflet.map.on("contextmenu", (e) => {
                this.leafletPlaceDestinationMarker(e.latlng);
            });
            this.leaflet.verticeRenderer = L.canvas({ padding: 0.5 });
        },
        teardownLeaflet() {
            this.leaflet.map.off();
            this.leaflet.map.remove();
        },
        leafletPlaceOriginMarker(latlng) {
            this.placeLeafletMarker(
                this.leaflet.originMarker,
                require("@/assets/blue-marker.png"),
                latlng
            );
        },
        leafletPlaceDestinationMarker(latlng) {
            this.placeLeafletMarker(
                this.leaflet.destinationMarker,
                require("@/assets/red-marker.png"),
                latlng
            );
        },
        drawLeafletLine(path, waypoints, color = "green") {
            path?.remove();
            path = new L.Polyline(waypoints, {
                color,
                weight: 3,
                opacity: 0.5,
                smoothFactor: 1,
            });
            path.addTo(this.leaflet.map);
            return path;
        },
        placeLeafletMarker(marker, iconUrl, latlng) {
            marker.setLatLng(latlng).addTo(this.leaflet.map);
            marker.dragging.enable();
            marker.setIcon(L.icon({ iconUrl }));
        },
        cesiumShowGraphNodes(vertices) {
            try {
                this.cesium.graphVerticeMarker =
                    this.cesium.viewer.scene.primitives.add(
                        new Cesium.PointPrimitiveCollection()
                    );
                vertices.forEach((v) =>
                    this.cesium.graphVerticeMarker.add({
                        position: Cesium.Cartesian3.fromDegrees(v.lon, v.lat),
                        color: Cesium.Color.RED,
                    })
                );
            } catch (e) {
                console.error(e);
            }
        },
        cesiumHideGraphNodes() {
            if (this.cesium.graphVerticeMarker)
                this.cesium.viewer.scene.primitives.remove(
                    this.cesium.graphVerticeMarker
                );
            this.cesium.graphVerticeMarker = null;
        },
        leafletShowGraphNodes(vertices) {
            try {
                let vtx = vertices.map((v) =>
                    L.circleMarker([v.lat, v.lon], {
                        renderer: this.leaflet.verticeRenderer,
                        color: "#f03",
                        radius: 1,
                    })
                );
                this.leaflet.graphVerticeMarker = L.layerGroup(vtx);
                this.leaflet.graphVerticeMarker.addTo(this.leaflet.map);
            } catch (e) {
                console.error(e);
            }
        },
        leafletHideGraphNodes() {
            if (this.leaflet.graphVerticeMarker)
                this.leaflet.graphVerticeMarker.remove();
            this.leaflet.graphVerticeMarker = null;
        },
        cesiumShowSearchSpace(vertices) {
            try {
                this.cesium.searchSpaceVerticeMarker =
                    this.cesium.viewer.scene.primitives.add(
                        new Cesium.PointPrimitiveCollection()
                    );
                vertices.forEach((v) =>
                    this.cesium.searchSpaceVerticeMarker.add({
                        position: Cesium.Cartesian3.fromDegrees(v.lon, v.lat),
                        color: Cesium.Color.CYAN,
                        pixelSize: 3,
                    })
                );
            } catch (e) {
                console.error(e);
            }
        },
        cesiumHideSearchSpace() {
            if (this.cesium.searchSpaceVerticeMarker)
                this.cesium.viewer.scene.primitives.remove(
                    this.cesium.searchSpaceVerticeMarker
                );
            this.cesium.searchSpaceVerticeMarker = null;
        },
        leafletShowSearchSpace(vertices) {
            try {
                let vtx = vertices.map((v) =>
                    L.circleMarker([v.lat, v.lon], {
                        renderer: this.leaflet.verticeRenderer,
                        color: "#0000ff",
                        radius: 1,
                    })
                );
                this.leaflet.searchSpaceMarker = L.layerGroup(vtx);
                this.leaflet.searchSpaceMarker.addTo(this.leaflet.map);
            } catch (e) {
                console.error(e);
            }
        },
        leafletHideSearchSpace() {
            if (this.leaflet.searchSpaceMarker)
                this.leaflet.searchSpaceMarker.remove();
            this.leaflet.searchSpaceMarker = null;
        },
    },
};
</script>

<style scoped>
#leafletMap {
    width: 100%;
    height: 560px;
    z-index: 0;
}

#cesiumContainer {
    width: 100%;
    height: 560px;
}
</style>
