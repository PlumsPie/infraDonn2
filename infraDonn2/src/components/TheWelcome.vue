
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
  post_likes:BigInteger
  attributes: {
    creation_date: any
  }
}
declare interface Comment {
   _id: string;
  _rev: string;
  post_id: string
  comment_content: string
  comment_author:string
  attributes: {
    creation_date: any
  }
}

// Référence à la base de données
const storage = ref();
const storageComments = ref();
let offline = ref(false);
const sync = ref();
const syncComments = ref();
const needle = ref();
const needleComment = ref();

// Données stockées
const postsData = ref<Post[]>([])
const commentsData = ref<Comment[]>([])

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  //fetchData()

})


// Initialisation de la base de données
  //  const db = new PouchDB('http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db')

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
   const db = new PouchDB('Posts');
   const dbComments = new PouchDB('Comments');
  storage.value = db;
  storageComments.value = dbComments;

  if (!db || !dbComments) {
    console.warn("Échec lors de la connexion aux bases");
    return;
  }


    console.log('Connecté à la collection : ' + db?.name + 'et la collection : ' + dbComments?.name);

    storage.value.createIndex({
      index: {
        fields: ['post_content']
      }
    }).then(console.log("the index has been created!"));


    storage.value.replicate.from("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db")
    fetchData()
    if (!offline.value) {
      startSync();
    }

    storageComments.value.replicate.from("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db_comments")
    fetchData()
    if (!offline.value) {
      startSync();
    }


}

const startSync = () => {
  if (sync.value) {
    console.log("Posts syncing is already running");
    return;
  }
  if (!storage.value) {
    console.warn('No local DB to sync');
    return;
  }
  if (syncComments.value) {
    console.log("Comments syncing is already running");
    return;
  }
  if (!storageComments.value) {
    console.warn('No local DB to sync');
    return;
  }

  sync.value = storage.value.sync("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db",
    {
      live: true,
      retry: true
    })
    .on('change', (info: any) => {
      console.log("Sync change", info);
      fetchData();
    })


    syncComments.value = storageComments.value.sync("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/infradonn_db_comments",
    {
      live: true,
      retry: true
    })
    .on('change', (info: any) => {
      console.log("Sync change", info);
      fetchData();
    })
  console.log('Sync started');
}


const stopSync = () => {
  try {
    sync.value.cancel()
    sync.value = null
    syncComments.value.cancel()
    syncComments.value = null

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
      postsData.value = result.rows.map((row: any) => row.doc).filter(p => !(p._id.startsWith("_design")));
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des posts :', error)
    });
    storageComments.value.allDocs({
    include_docs: true

  }).then((result: any) => {
    console.log('=>Données récuperées', result.rows);
    commentsData.value = result.rows.map((row: any) => row.doc);

  }).catch(function (err: any) {
    console.error('=> Erreur lors de la récupération des commentaires :', err)
  });

}
const doc = {
  post_name: 'Super Document',
  post_content: 'SuperContenu',
  post_likes:0,
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
const com ={
  comment_author:`Quelqu'un de très chouette sans doute`,
  comment_content: 'Super commentaire',
  comment_likes:0
}

const addDoc = (doc: any) => {
  storage.value
    .post(doc)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const addCom = (post_id: any, com:any) => {
   if (!storageComments.value) {
    console.warn("La collection comments n'existe pas");
    return;
  }
  com.post_id=post_id
  storageComments.value
    .post(com)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const updateDoc = (doc: any) => {
  doc.post_name= needle.value;
  needle.value = '';
  storage.value
    .put(doc)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const updateCom = (com:any) => {
  com.comment_content= needleComment.value;
  needleComment.value = '';
  storageComments.value
    .post(com)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}

const likePost = (doc:any) => {
  doc.post_likes++;
  storage.value
    .put(doc)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}

const deleteDoc = (docId: any, docRev: any) => {

  storage.value
    .remove(docId, docRev)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const deleteCom = (comId: any, comRev: any) => {

  storageComments.value
    .remove(comId, comRev)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const creat100Docs = (count = 100) => {
  const words = ["J'aime les bananes", "Le printemps est clairement la pire saison", "Si tu lis ce message, envoie-le à 15 de tes contacts ou tu mourras",
   "Vous utilisez quoi comme crème après-rasage?", "Roses are red, violets are blue, if you know how to end this poem, then go ahead", "I'm quitting.", "Go away!"];

  for (let i = 0; i < count; i++) {
    storage.value.post({
      post_name: "Auto-Document " + i,
      post_content: words[Math.floor(Math.random() * words.length)]
    });
  }

  fetchData();
};

const getComments = (postId: any)=>{
const commentsFiltered = commentsData.value.filter(a => a.post_id === postId)
  return commentsFiltered;
}

const search = (event: any) => {
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
  <button @click="creat100Docs()">Générer 100 documents</button>
  <input type="text" placeholder="Search" @keyup.enter="search" class="search">
  <br>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <hr/>
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
    <p>{{ post.post_likes }}</p>

    <label for="needle">Editer le post</label>
    <input type="text" name="needle" v-model="needle" @blur="updateDoc(post)">


    <button @click="deleteDoc(post._id,post._rev)">Supprimer Super Document</button>
    <button @click="likePost(post)">J'aime!</button>
    <button @click="addCom(post._id, com)">Commenter</button>

    <article v-for="comment in getComments(post._id)" v-bind:key="(comment as any).id">

          <ul>
            <li>
              <h2>{{ comment.comment_content }}</h2>

              <label for="needleComments">Editer le commentaire</label>
              <input type="text" name="needleComments" v-model="needleComment" @blur="updateCom(comment)">
              <button @click="deleteCom(comment._id, comment._rev)">Supprimer le commentaire</button>
            </li>
          </ul>

        </article>

  </article>
  <button @click="replicateDB()">Valider les changements</button>

</template>
