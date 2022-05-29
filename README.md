# osm-ship-routing-gui

Using Docker is the fastest way to get the frontend of the OSM-Ship-Routing running.
You also need the [backend of the OSM-Ship Routing Service](https://github.com/dmholtz/osm-ship-routing) to calculate ship routes.

## Description

### General

This webapp shows a map where you can select the `origin` and `destination` by left- or right-clicking on the map.\
If the `origin` and `destination` is selected, you can click on the `Navigate` button to compute the shipping route. If there is none available (e.g. because you selected two locations on the land or because the locations are not connected by water) it will display so. Otherwise, you will see the computed route as a white line on the map and see the distance in the field under the button.

### Note

-   The Mediterranean may not be connected to the ocean, so there is no route between two places there.
-   You can switch between Leaflet and Cesium as map provider.
-   Sometimes, Cesium lags a bit. Then the computed routes or the earth need some time to render.
-   If possible, a red line displays the direct connection (using Great Circle) and a green line display a rhumb line between the origin and destination.
-   You can switch between a spherical map and a planar projection when using Cesium as map provider.

## Setup Using Docker

1. Pull the image from [Dockerhub](https://hub.docker.com/repository/docker/natevvv/osm-ship-routing-gui): `docker pull natevvv/osm-ship-routing-gui:<TAG>`
2. Start a container: `docker run -p 8080:8080 --name osm-gui natevvv/osm-ship-routing-gui`
3. Access the GUI via a webbrowser: http://localhost:8080

Note that `<TAG>` needs to be replaced by a valid tag. Please find all available tags on [Dockerhub](https://hub.docker.com/repository/docker/natevvv/osm-ship-routing-gui/tags?page=1&ordering=last_updated).
Tag `1.0.0` refers to the first submission and tag `latest` referst to the most recent release on Dockerhub.

## Project setup / Installation from source

The `osm-ship-routing-gui` is written in [Node.js](https://nodejs.org/en/), using version `v16.15.0`.
To install this version, you can use [nvm](https://github.com/nvm-sh/nvm).\
Additionally, this project uses [Vue.js](https://vuejs.org/guide/quick-start.html#with-build-tools), [Vuetify](https://vuetifyjs.com/en/getting-started/installation/) (and [vue-cli](https://cli.vuejs.org/guide/installation.html))

### Install project

```
npm install
```

### Compiles and hot-reloads for development

```
npm run serve
```

### Compiles and minifies for production

```
npm run build
```

### Lints and fixes files

```
npm run lint
```

### Customize configuration

See [Configuration Reference](https://cli.vuejs.org/config/).
