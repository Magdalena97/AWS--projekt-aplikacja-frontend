<template>
    <v-card class="d-inline-block mx-auto my-2" :color="backgroundColor">
        <v-container>
            <v-row justify="space-between">
                <v-col cols="auto">
                    <v-img height="200" width="200" :src="imgSrc"></v-img>
                </v-col>
                <v-col cols="auto" class="text-center pl-0">
                    <v-row class="flex-column ma-0 fill-height" justify="center">
                        <v-col class="px-0">
                            <v-checkbox v-if="!object.sent" class="ml-1" v-model="object.selected"
                                @click="changeSelect" />
                            <v-btn v-else icon>
                                <v-icon color="success">mdi-check-circle</v-icon>
                            </v-btn>
                        </v-col>

                        <v-col class="px-0">
                            <v-btn icon>
                                <v-icon>mdi-share-variant</v-icon>
                            </v-btn>
                        </v-col>
                    </v-row>
                </v-col>
            </v-row>
        </v-container>
    </v-card>
</template>

<script>
export default {
    props: {
        object: {
            type: Object,
            required: true
        },
        bucketName: {
            type: String,
            required: true,
        }
    },
    data: () => ({
        backgroundColor: ''
    }),
    computed: {
        imgSrc() {
            return 'https://'+this.bucketName+'.s3.amazonaws.com/' + this.object.Key
        }
    },
    methods: {
        changeSelect() {
            this.backgroundColor = (this.object.selected) ? 'grey lighten-2' : '';
        }
    }
}
</script>

<style>

</style>