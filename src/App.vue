<template>
    <v-app>
        <v-app-bar app color="primary" dark>
            <div class="d-flex align-center">
                Lab Course Algorithmics: Ship Routing
            </div>

            <v-spacer></v-spacer>

            <v-btn
                href="https://github.com/NateVVV/osm-ship-routing-gui"
                target="_blank"
                text
            >
                <span class="mr-2">Release / Code Infromation</span>
                <v-icon>mdi-open-in-new</v-icon>
            </v-btn>
        </v-app-bar>

        <v-main>
            <ShipRouting />
        </v-main>
    </v-app>
</template>

<script>
//import HelloWorld from "./components/HelloWorld";
import ShipRouting from "./components/ShipRouting";

export default {
    name: "App",

    components: {
        ShipRouting,
    },

    data: () => ({
        //
    }),
};
</script>
