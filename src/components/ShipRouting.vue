<template>
    <v-container>
        <div id="cesiumContainer"></div>
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
/* global Cesium */
/* es-lint no-undef: "error" */
import axios from "axios";

export default {
    name: "ShipRouting",
    data: () => ({
        path: { waypoints: null, length: 0 },
        navigating: false,
        viewer: null,
        cesiumPath: null,
        cesiumGreatCircle: null,
        cesiumRhumbLine: null,
        cesiumOriginMarker: null,
        cesiumDestinationMarker: null,
    }),
    computed: {
        origin() {
            if (!this.cesiumOriginMarker) return null;
            const cartographic = Cesium.Cartographic.fromCartesian(
                this.cesiumOriginMarker._position._value
            );
            return {
                lat: Cesium.Math.toDegrees(cartographic.latitude),
                lng: Cesium.Math.toDegrees(cartographic.longitude),
            };
        },
        destination() {
            if (!this.cesiumDestinationMarker) return null;
            const cartographic = Cesium.Cartographic.fromCartesian(
                this.cesiumDestinationMarker._position._value
            );
            return {
                lat: Cesium.Math.toDegrees(cartographic.latitude),
                lng: Cesium.Math.toDegrees(cartographic.longitude),
            };
        },
    },
    mounted() {
        this.setupCesium();
    },
    methods: {
        async computeRoute() {
            this.navigating = true;
            const waypoints = [this.origin, this.destination];
            const degreesArray = waypoints.map((w) => [w.lng, w.lat]).flat();
            this.viewer.entities.remove(this.cesiumRhumbLine);
            this.viewer.entities.remove(this.cesiumPath);
            this.viewer.entities.remove(this.cesiumGreatCircle);
            this.cesiumRhumbLine = this.viewer.entities.add({
                name: "Green rhumb line on terrain",
                polyline: {
                    positions: Cesium.Cartesian3.fromDegreesArray(degreesArray),
                    width: 3,
                    material: Cesium.Color.GREEN,
                    arcType: Cesium.ArcType.RHUMB,
                    clampToGround: true,
                },
            });
            this.cesiumGreatCircle = this.viewer.entities.add({
                name: "Red line on terrain",
                polyline: {
                    positions: Cesium.Cartesian3.fromDegreesArray(degreesArray),
                    width: 3,
                    material: Cesium.Color.RED,
                    clampToGround: true,
                },
            });
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
                const degreesArray = waypoints
                    .map((w) => [w.lon, w.lat])
                    .flat();
                this.cesiumPath = this.viewer.entities.add({
                    name: "Ship route",
                    polyline: {
                        positions:
                            Cesium.Cartesian3.fromDegreesArray(degreesArray),
                        width: 3,
                        material: Cesium.Color.WHITE,
                        clampToGround: true,
                    },
                });
                console.log(data);
            } catch (e) {
                console.error(e);
            }
            this.navigating = false;
        },
        setupCesium() {
            Cesium.Ion.defaultAccessToken =
                "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiIxMDk0MThjNS0yODUyLTRiYTItOTc0MC0wNjhjZmJkNWFmYjQiLCJpZCI6OTQ0NjYsImlhdCI6MTY1Mjk2NTE2Mn0.fyZexWFaRBx0YQqQEJnxL-BtmObI3uXfxcPvNEi7LFg";

            const extent = Cesium.Rectangle.fromDegrees(-1, 19, 3, 17);

            Cesium.Camera.DEFAULT_VIEW_RECTANGLE = extent;
            Cesium.Camera.DEFAULT_VIEW_FACTOR = 3;

            // Initialize the Cesium Viewer in the HTML element with the "cesiumContainer" ID.
            this.viewer = new Cesium.Viewer("cesiumContainer", {
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

            const scene = this.viewer.scene;
            if (!scene.pickPositionSupported) {
                window.alert("This browser does not support pickPosition.");
            }
            // Mouse over the globe to see the cartographic position
            const handler = new Cesium.ScreenSpaceEventHandler(scene.canvas);
            handler.setInputAction((movement) => {
                let position = this.viewer.scene.camera.pickEllipsoid(
                    movement.position,
                    this.viewer.scene.globe.ellipsoid
                );
                if (!position) {
                    console.log("Globe was not picked");
                    return;
                }
                let cartographic = Cesium.Cartographic.fromCartesian(position);
                let lat = Cesium.Math.toDegrees(cartographic.latitude);
                let lng = Cesium.Math.toDegrees(cartographic.longitude);
                if (this.cesiumOriginMarker)
                    this.viewer.entities.remove(this.cesiumOriginMarker);
                this.cesiumOriginMarker = this.viewer.entities.add({
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
                let position = this.viewer.scene.camera.pickEllipsoid(
                    movement.position,
                    this.viewer.scene.globe.ellipsoid
                );
                if (!position) {
                    console.log("Globe was not picked");
                    return;
                }
                let cartographic = Cesium.Cartographic.fromCartesian(position);
                let lat = Cesium.Math.toDegrees(cartographic.latitude);
                let lng = Cesium.Math.toDegrees(cartographic.longitude);
                if (this.cesiumDestinationMarker)
                    this.viewer.entities.remove(this.cesiumDestinationMarker);
                this.cesiumDestinationMarker = this.viewer.entities.add({
                    position: Cesium.Cartesian3.fromDegrees(lng, lat),
                    billboard: {
                        image: require("@/assets/red-marker.png"),
                        //pixelOffset: new Cesium.Cartesian2(0, -50), // default: (0, 0)
                        horizontalOrigin: Cesium.HorizontalOrigin.CENTER, // default
                        verticalOrigin: Cesium.VerticalOrigin.BOTTOM, // default: CENTER
                        scale: 0.3,
                        //disableDepthTestDistance: Number.POSITIVE_INFINITY, // necessary when using Cesium World Terrain because of rendering issues
                    },
                });
            }, Cesium.ScreenSpaceEventType.RIGHT_CLICK);
        },
    },
};
</script>

<style scoped>
#map {
    width: 600px;
    height: 500px;
}

.v-progress-circular {
    margin: 1rem;
}
</style>
