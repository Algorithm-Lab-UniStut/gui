<template>
    <v-container>
        <div>
            <v-switch v-model="leafletSwitch" label="Leaflet"></v-switch>
            <v-switch v-model="cesiumSwitch" label="Cesium"></v-switch>
        </div>
        <div id="leafletMap" v-if="leafletSwitch"></div>
        <div id="cesiumContainer" v-if="cesiumSwitch"></div>
        <div>
            <v-banner>
                <v-icon color="blue darken-2" small>mdi-map-marker</v-icon
                >Origin:
                {{ origin ? `${origin.lat}, ${origin.lng}` : "" }}
            </v-banner>
            <v-banner>
                <v-icon color="red darken-2" small>mdi-map-marker</v-icon
                >Destination:
                {{
                    destination ? `${destination.lat}, ${destination.lng}` : ""
                }}
            </v-banner>
        </div>
        <div>
            <v-btn
                :color="navigating ? 'light-grey' : 'primary'"
                block
                id="navigateBtn"
                :disabled="navigating || !origin || !destination"
                @click="computeRoute"
            >
                <v-progress-circular
                    v-if="navigating"
                    indeterminate
                    color="grey"
                ></v-progress-circular>
                <div v-else>Navigate</div>
            </v-btn>
            <v-banner class="pa-2 my-4" v-if="path.length">
                Length: {{ path.length / 1000 }} km</v-banner
            >
        </div>
    </v-container>
</template>

<script>
/* global Cesium, L */
/* es-lint no-undef: "error" */
import axios from "axios";

export default {
    name: "ShipRouting",
    data: () => ({
        path: { waypoints: null, length: 0 },
        navigating: false,
        cesiumSwitch: true,
        leafletSwitch: true,
        leaflet: {
            map: null,
            originMarker: L.marker(),
            destinationMarker: L.marker(),
            setOrigin: 1,
            path: null,
        },
        cesium: {
            viewer: null,
            greatCircleLine: null,
            rhumbLine: null,
            originMarker: null,
            destinationMarker: null,
            path: null,
        },
    }),
    computed: {
        origin() {
            if (!this.cesium.originMarker) return null;
            const cartographic = Cesium.Cartographic.fromCartesian(
                this.cesium.originMarker._position._value
            );
            return {
                lat: Cesium.Math.toDegrees(cartographic.latitude),
                lng: Cesium.Math.toDegrees(cartographic.longitude),
            };
        },
        destination() {
            if (!this.cesium.destinationMarker) return null;
            const cartographic = Cesium.Cartographic.fromCartesian(
                this.cesium.destinationMarker._position._value
            );
            return {
                lat: Cesium.Math.toDegrees(cartographic.latitude),
                lng: Cesium.Math.toDegrees(cartographic.longitude),
            };
        },
    },
    watch: {
        async leafletSwitch() {
            this.cesiumSwitch = !this.leafletSwitch;
            if (this.leafletSwitch) {
                await this.$nextTick();
                this.setupLeaflet();
            } else {
                this.teardownLeaflet();
            }
        },
        async cesiumSwitch() {
            this.leafletSwitch = !this.cesiumSwitch;
            if (this.cesiumSwitch) {
                await this.$nextTick();
                this.setupCesium();
            } else {
                this.teardownCesium();
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
            try {
                const { data } = await axios.post(
                    "http://localhost:8081/routes",
                    JSON.stringify({
                        origin: { lon: this.origin.lng, lat: this.origin.lat },
                        destination: {
                            lon: this.destination.lng,
                            lat: this.destination.lat,
                        },
                    })
                );
                const waypoints = data.path.waypoints;
                this.path.waypoints = waypoints;
                this.path.length = data.path.length;
                const waypointsDegreesArray = waypoints
                    .map((w) => [w.lon, w.lat])
                    .flat();
                if (this.cesiumSwitch) {
                    this.cesium.viewer.entities.remove(this.cesium.path);
                    this.cesium.path = this.cesium.viewer.entities.add({
                        name: "Ship route",
                        polyline: {
                            positions: Cesium.Cartesian3.fromDegreesArray(
                                waypointsDegreesArray
                            ),
                            width: 3,
                            material: Cesium.Color.WHITE,
                            clampToGround: true,
                        },
                    });
                }
                console.log(data);
            } catch (e) {
                console.error(e);
            } finally {
                const directPath = [this.origin, this.destination];
                const directPathDegreesArray = directPath
                    .map((w) => [w.lng, w.lat])
                    .flat();
                if (this.cesiumSwitch) {
                    this.cesium.viewer.entities.remove(this.cesium.rhumbLine);
                    this.cesium.viewer.entities.remove(
                        this.cesium.greatCircleLine
                    );
                    this.cesium.rhumbLine = this.cesium.viewer.entities.add({
                        name: "Green rhumb line on terrain",
                        polyline: {
                            positions: Cesium.Cartesian3.fromDegreesArray(
                                directPathDegreesArray
                            ),
                            width: 3,
                            material: Cesium.Color.GREEN,
                            arcType: Cesium.ArcType.RHUMB,
                            clampToGround: true,
                        },
                    });
                    this.cesium.greatCircleLine =
                        this.cesium.viewer.entities.add({
                            name: "Red line on terrain",
                            polyline: {
                                positions: Cesium.Cartesian3.fromDegreesArray(
                                    directPathDegreesArray
                                ),
                                width: 3,
                                material: Cesium.Color.RED,
                                clampToGround: true,
                            },
                        });
                }
                if (this.leafletSwitch) {
                    this.drawLeafletLine(directPath);
                }
                this.navigating = false;
            }
        },
        setupCesium() {
            Cesium.Ion.defaultAccessToken =
                "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiIxMDk0MThjNS0yODUyLTRiYTItOTc0MC0wNjhjZmJkNWFmYjQiLCJpZCI6OTQ0NjYsImlhdCI6MTY1Mjk2NTE2Mn0.fyZexWFaRBx0YQqQEJnxL-BtmObI3uXfxcPvNEi7LFg";

            const extent = Cesium.Rectangle.fromDegrees(-1, 19, 3, 17);

            Cesium.Camera.DEFAULT_VIEW_RECTANGLE = extent;
            Cesium.Camera.DEFAULT_VIEW_FACTOR = 3;

            // Initialize the Cesium Viewer in the HTML element with the "cesiumContainer" ID.
            this.cesium.viewer = new Cesium.Viewer("cesiumContainer", {
                //terrainProvider: Cesium.createWorldTerrain(),
                imageryProvider: Cesium.createWorldImagery({
                    style: Cesium.IonWorldImageryStyle.AERIAL_WITH_LABELS,
                }),
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
        drawCesiumLine(waypoints, type, name, color, width) {
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
            this.leaflet.map = L.map("leafletMap").setView([0, 0], 1);
            const accessToken =
                "pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw";
            L.tileLayer(
                `https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token=${accessToken}`,
                {
                    maxZoom: 18,
                    attribution:
                        'Map data &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors, ' +
                        'Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
                    id: "mapbox/streets-v11",
                    tileSize: 512,
                    zoomOffset: -1,
                }
            ).addTo(this.leaflet.map);
            this.leaflet.originMarker = L.marker();
            this.leaflet.destinationMarker = L.marker();
            this.setOrigin = 1;
            this.leaflet.map.on("click", this.onLeafletMapClick);
        },
        teardownLeaflet() {
            this.leaflet.map.off();
            this.leaflet.map.remove();
        },
        onLeafletMapClick(e) {
            let location =
                this.leaflet.setOrigin == 1
                    ? this.leaflet.originMarker
                    : this.leaflet.destinationMarker;
            let icon =
                this.leaflet.setOrigin == 1
                    ? L.icon({
                          iconUrl: require("@/assets/blue-marker.png"),
                      })
                    : L.icon({
                          iconUrl: require("@/assets/red-marker.png"),
                      });
            this.leaflet.setOrigin = (this.leaflet.setOrigin + 1) % 2;
            console.log("Clicked", e);
            location.setLatLng(e.latlng).addTo(this.leaflet.map);
            location.dragging.enable();
            location.setIcon(icon);
        },
        drawLeafletLine(waypoints) {
            this.leaflet.path?.remove();
            this.leaflet.path = new L.Polyline(waypoints, {
                color: "red",
                weight: 3,
                opacity: 0.5,
                smoothFactor: 1,
            });
            this.leaflet.path.addTo(this.leaflet.map);
        },
    },
};
</script>

<style scoped>
#leafletMap {
    width: 600px;
    height: 500px;
}

.v-progress-circular {
    margin: 1rem;
}
</style>