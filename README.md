# osm-ship-routing-gui

Using Docker is the fastest way to get the frontend of the OSM-Ship-Routing running.
You also need the [backend of the OSM-Ship Routing Service](https://github.com/dmholtz/osm-ship-routing) to calculate ship routes.

## Setup Using Docker

1. Pull the image from [Dockerhub](): `docker pull TODO:<TAG>`
2. Start a container: `docker run -p 8080:8080 --name osm-gui TODO`
3. Access the GUI via a webbrowser: http://localhost:8080

Note that `<TAG>` needs to be replaced by a valid tag. Please find all available tags on [Dockerhub]().
Tag `1.0.0` refers to the first submission and tag `latest` referst to the most recent release on Dockerhub.

## Project setup / Installation from source
The `osm-ship-routing-gui` is written in [Node.js](https://nodejs.org/en/), using version `v16.15.0`.
To install this version, you can use [nvm](https://github.com/nvm-sh/nvm).\
Additionally, this project uses [Vue.js](https://vuejs.org/guide/quick-start.html#with-build-tools), [Vuetify](https://vuetifyjs.com/en/getting-started/installation/) (and [vue-cli](https://cli.vuejs.org/guide/installation.html))

To install the components, run: 
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
