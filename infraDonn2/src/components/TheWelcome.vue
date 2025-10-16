<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  post_name: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db')
  if (db) {
    console.log('Connecté à la collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

// Récupération des données
const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}
const doc = {
  post_name: 'SuperPost',
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

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
  addDoc(doc)
})
</script>

<template>
  <h1>Fetch Data</h1>
  <button @click="addDoc(doc)">Nouveau Document</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
</template>

<!-- /* <script setup lang="ts">
import { ref } from 'vue'
import { onMounted } from 'vue'
import PouchDB from 'pouchdb'
// npm run dev to restart server
// DATA - MODEL
const counter = ref(20)
const storage = ref()

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db')
  if (db) {
    console.log('Connected to collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}
// METHODS - CONTROLLER
const increment = () => {
  counter.value++
}
</script>

<template>
  <h1>Hello World</h1>
  <button @click="increment">Increment</button>
  <p style="color: white; font-size: 50px">{{ counter }}</p>
</template>
*/ -->
