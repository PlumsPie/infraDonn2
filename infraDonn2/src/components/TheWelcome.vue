
<script setup lang="ts">
// npm run dev to restart server dans C:\Users\sarah\OneDrive\Documents\05_HEIG\S3\InfraDon2\CodeBase\infraDonn2\infraDonn2
// connexion couch DB: http://127.0.0.1:5984/_utils/#login
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind);
declare interface Post {
   _id: string;
  _rev: string;
  post_name: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

// Référence à la base de données
const storage = ref()
let offline = ref(false);
const sync = ref();
// Données stockées
const postsData = ref<Post[]>([])

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()

})


// Initialisation de la base de données
  //  const db = new PouchDB('http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db')

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
   const db = new PouchDB('Posts');
  if (db) {
    console.log('Connecté à la collection : ' + db?.name);
    storage.value = db;
    storage.value.createIndex({
      index: {
        fields: ['post_content']
      }
    }).then(console.log("the index has been created!"));

    storage.value.replicate.from("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2")
    fetchData()
    if (!offline.value) {
      startSync();
    }

  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

const startSync = () => {
  if (sync.value) {
    console.log("sync already running");
    return;
  }
  if (!storage.value) {
    console.warn('No local DB to sync');
    return;
  }

  sync.value = storage.value.sync("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2",
    {
      live: true,
      retry: true
    })
    .on('change', (info: any) => {
      console.log("sync change", info);
      fetchData();
    })

  console.log('Sync started');
}


const stopSync = () => {
  try {
    sync.value.cancel()
    sync.value = null
    console.log('Sync cancelled')
  } catch (err) {
    console.error('Error cancelling sync', err)
  }
}


const handleChange = () => {

  if (!offline.value) {
    console.log("Mode ONLINE => lancement du sync");
    startSync();
  } else {
    console.log("Mode OFFLINE → arrêt du sync");
    stopSync();
  }
}

const replicateDB = ()=>{
  storage.value.replicate.to("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db")
}

// Récupération des données
const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {

      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}
const doc = {
  post_name: 'Super Document',
  post_content: 'SuperContenu',
  comments: [
    {
      title: 'Hello',
      author: 'Alice',
    },
    {
      title: 'Hi',
      author: 'Bob',
    },
  ],
}

const addDoc = (doc) => {
  storage.value
    .post(doc)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const updateDoc = (docId, docRev) => {
  storage.value
    .put({
      _id:docId,
      _rev:docRev,
      post_name: 'Super Document Modifié'
    })
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const deleteDoc = (docId, docRev) => {

  storage.value
    .remove(docId, docRev)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const generate200Docs = (count = 200) => {
  const words = ["léa", "inoé", "camilo", "yannis", "sarah", "tanguy", "dylan"];

  for (let i = 0; i < count; i++) {
    storage.value.post({
      title: "Doc " + i,
      post_content: words[Math.floor(Math.random() * words.length)]
    });
  }

  fetchData();
};



const search = (event: Event) => {
  const value = event.target.value.trim();

  if (!value) {
    return fetchData();
  }

  storage.value.find({
    selector: { post_content: event.target.value }
  })

    .then((result: any) => {
      console.log('=> Données récupérées :', result.docs)
      postsData.value = result.docs;

    })
    .catch((error: any) => {
      console.error(error)
    })
}


</script>

<template>
  <h1>Fetch Data</h1>
  <p>Mode offline {{ offline }}</p>
  <label for="mode" name="mode">Mode offline</label>
  <input type="checkbox" id="mode" name="mode" v-model="offline" @change="handleChange">
<br>
  <button @click="addDoc(doc)">Nouveau Super Document</button>
  <button @click="generate200Docs()">Générer 200 documents</button>
  <input type="text" placeholder="Search" @keyup.enter="search" class="search">
  <br>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <hr/>
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>

    <button @click="deleteDoc(post._id,post._rev)">Supprimer Super Document</button>
    <button @click="updateDoc(post._id,post._rev)">Editer Super Document</button>
  </article>
  <button @click="replicateDB()">Valider les changements</button>

</template>
