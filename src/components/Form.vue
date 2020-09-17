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
        <div class="d-flex flex-wrap justify-center align-content-center"><!--  moje obrazki wysweltnoe w petli  image oject to komponret z pojedynczym obzkiem-->
            <ImageObject v-for="(object) in bucket" :key="object.id" :object="object" :bucketName="bucketName"/>
        </div>
    </v-container>
</template>

<script>
import ImageObject from './ImageObject';

const {
    VUE_APP_S3_BUCKET_NAME
} = process.env

export default {
    name: 'Form',
    components: {
        ImageObject
    },
    data: () => ({ //--  data to moje lista zmiennych
        file: null, //to moj obrazek
        showMessage: false,
        message: '',//widomosc u gory ekranu
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
        submit() {//reakcje na iterakcje z uzytkowniimem jak uztrkownik nacisnie przcisk wysljj ZIELONY WYSLIJ
            if (this.file) { 
                var self = this
                fetch('http://localhost:3000/getPresignedPost')// wysylam zapytanie s=do serear by dał mi sygnatury
                .then(response => response.json())//jak odbrze sygnatury
                .then((data) => { //to jest ta moja odpiwdz z serera z sygnatura
                        const formData = new FormData();// dostała juz synature i musze plik wysłac do s3
                        formData.append("Content-Type", self.file.type);
                        Object.entries(data.fields).forEach(([k, v]) => {
                            formData.append(k, v);
                        });
                        formData.append("file", self.file); // must be the last one
                        fetch(data.url, {//wysyła mi oj obrazek do s3
                            method: "POST",
                            body: formData,
                        }).then(() => {//jak wysle dane to wywoluje sie then
                            self.file = null// w formularzu nie ma byc juz załadowany plik
                            self.showMessage = true;
                            self.messageType = 'success';
                            self.message = "File was uploaded! Please refresh S3 files list.";
                        });
                }).catch((e) => {//jak nie ua mi sie skontaktowac z sewreem
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
        sendList() {//Wyslij do kolejki - naciskamyu ten przycsk.. pliki sa wysylane najpierw do serear a potem do kolejki 
        //najpierw zanczone obrazki da wysylane do serera a potem serwer wysyła je do sqs bysmy mogli je przetwrzyc i dac im znak wodny
            var selected = this.bucket.filter((item) => item.selected);// wyławiamyu zaznaczone obrazki
            if (selected.length==0) {
                this.showMessage = true;
                this.messageType = 'warning';
                this.message = "Error: No images selected!";
                return;
            } 
            this.loadingSendButton = true;
            
            var data = new FormData();
            data.append("json", JSON.stringify(selected.map(item => item.Key)));
            fetch("http://localhost:3000/sendMessages",//koljen zapytanie do serera 
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
            //to zielone gowni i nie mozemy ich jeszcze raz wysłac do koeljki
                if (selected.includes(item)) {
                    item.id = item.Key + Math.floor(Math.random() * 100)
                    item.selected = false;
                    item.sent = true;
                }
                return item;
            }));
            this.loadingSendButton = false;
        },
        updateList() {//naciskamy przycisk odswerz awartosc s3
            var self = this;
            (async function() {
                self.loadingUpdateButton = true;
                const response = fetch('http://localhost:3000/listOfObjects')// wysyłam zapytanie do serera by sciagnal liste plikow z s3
                .then(response => response.json())
                .then((data) => {
                    self.loadingUpdateButton = false;
                    if (data && data.Contents) {
                    self.bucket = data.Contents.filter((item) => (!item.Key.endsWith("/") && item.Key.startsWith("images/")));//filtrujemy ta liste by zawierala pliki w 
                    //katalogu images i zeby nie bac plikow zakonczonych na / bo to folder
                    var toDelete = [];// pusta tablica z rzeczami do wywalnia
                    self.bucket = self.bucket.map((item) => {//nowa tablica z obrazkami by pozbyc sie duplikacji
                        if (item.Key.endsWith('-watermark')) {//jezeli dany obrazek ma juz watermark to ten obrazem ma wyswieylic
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