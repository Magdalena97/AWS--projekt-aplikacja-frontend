<template>
    <v-container>
        <v-alert :color="messageType" v-if="!!message" dismissible v-model="showMessage">
            {{ message }}
        </v-alert>
        <v-form>
            <v-file-input name="image" v-model="file" placeholder="Dodaj plik" />
            <v-btn block color="success" @click="submit">Wyślij</v-btn>
        </v-form>
        <div class="d-flex justify-center">
            <v-btn class="my-3 mx-2" color="info" @click="updateList" :loading="loadingUpdateButton">Odśwież zawartość S3</v-btn>
            <v-btn class="my-3 mx-2" color="warning" @click="sendList" :loading="loadingSendButton">Wyślij do kolejki</v-btn>
        </div>
        <div class="d-flex flex-wrap justify-center align-content-center">
            <ImageObject v-for="(object) in bucket" :key="object.id" :object="object" :bucketName="bucketName"/>
        </div>
    </v-container>
</template>

<script>
import ImageObject from './ImageObject';

const {
    VUE_APP_S3_BUCKET_NAME,
    VUE_APP_CURRENT_URL
} = process.env

export default {
    name: 'Form',
    components: {
        ImageObject
    },
    //lista zmiennych
    data: () => ({ 
        file: null, 
        showMessage: false,
        message: '',
        bucket: [],
        loadingUpdateButton: false,
        loadingSendButton: false,
        messageType: 'warning'
    }),
    computed: {
        bucketName() {
            return VUE_APP_S3_BUCKET_NAME;
        },      
    },
    methods: {
        submit() {//Reakcja na nacisniecie przycisku WYŚLIJ
            if (this.file) { 
                var self = this
                //Wyslanie zapytanie do serwera o sygnature
                fetch('http://' + VUE_APP_CURRENT_URL+':3000/getPresignedPost')
                .then(response => response.json())//odebranie sygnatury
                .then((data) => { 
                        const formData = new FormData();
                        formData.append("Content-Type", self.file.type);
                        Object.entries(data.fields).forEach(([k, v]) => {
                            formData.append(k, v);
                        });
                        formData.append("file", self.file); 
                        fetch(data.url, {//wysyłanie obrazka do s3
                            method: "POST",
                            body: formData,
                        }).then(() => {
                            self.file = null
                            self.showMessage = true;
                            self.messageType = 'success';
                            self.message = "File was uploaded! Please refresh S3 files list.";
                        });
                }).catch((e) => {
                    console.log(e)
                    this.showMessage = true;
                    this.messageType = 'warning';
                    this.message = "Error: Cannot connection to server!";    
                });
            } else {
                this.showMessage = true;
                this.messageType = 'warning';
                this.message = "Error: No file selected!";
            }
        }, 
        //Reakcja na przycisk WYślij do kolejki
        sendList() {
            var selected = this.bucket.filter((item) => item.selected);
            if (selected.length==0) {
                this.showMessage = true;
                this.messageType = 'warning';
                this.message = "Error: No images selected!";
                return;
            } 
            this.loadingSendButton = true;
            
            var data = new FormData();
            data.append("json", JSON.stringify(selected.map(item => item.Key)));
            fetch('http://' + VUE_APP_CURRENT_URL+":3000/sendMessages",//zapytanie do serwera o wysłanie obrazków do kolejki
            {
                method: "POST",
                body: JSON.stringify(selected.map(item => item.Key)),
                headers: {
                    'Accept': 'application/json',
                    'Content-Type': 'application/json'
                },
            })
            .then(function(res){ return res.json(); })
            .then(function(data){ alert( JSON.stringify( data ) ) })
            this.bucket = this.bucket.map((item => {//odznaczanie obrazkow i oznaczanie ich ich jako wyslane - 
            //Zielony znak oznacza, że obrazek jest przetworzony
                if (selected.includes(item)) {
                    item.id = item.Key + Math.floor(Math.random() * 100)
                    item.selected = false;
                    item.sent = true;
                }
                return item;
            }));
            this.loadingSendButton = false;
        },
        updateList() {//Reakcja na przyscik odświerz zawartość s3
            var self = this;
            (async function() {
                self.loadingUpdateButton = true;
                const response = fetch('http://' + VUE_APP_CURRENT_URL+':3000/listOfObjects')//Zapytanie do serwera o pobranie listy obrazków
                .then(response => response.json())
                .then((data) => {
                    self.loadingUpdateButton = false;
                    if (data && data.Contents) { 
                    //odpowiednia filtracja obrazków
                    self.bucket = data.Contents.filter((item) => (!item.Key.endsWith("/") && item.Key.startsWith("images/")));
                    var toDelete = [];
                    self.bucket = self.bucket.map((item) => {
                        if (item.Key.endsWith('-watermark')) {//Wyswietlanie tylko obrazkow z watermark
                            toDelete.push(item.Key.replace('-watermark',''))
                            item.sent = true
                        } else {
                            item.sent = false
                        }
                        item.id = item.Key + Math.floor(Math.random() * 100)
                        item.selected = false;
                        return item;
                    }).filter((item) => !toDelete.includes(item.Key))
                    console.log(toDelete)
                }
                });
            })();
        }
    }
}

</script>

<style>

</style>