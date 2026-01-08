
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
  post_likes:number
  attributes: {
    creation_date: any
  }
   _attachments?: Record<
  string,
  {content_type:string; length?: number; digest?: string; stub?: boolean}
  >
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
const offline = ref(false);
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

})
const page = ref(0);

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
        fields: ['post_likes']
      }
    }).then(console.log("the index has been created!"));


    storage.value.replicate.from("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db")
    storageComments.value.replicate.from("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db_comments")

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

  sync.value = storage.value.sync("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db",
    {
      live: true,
      retry: true
    })
    .on('change', (info: any) => {
      console.log("Sync change", info);
      fetchData();
    })


    syncComments.value = storageComments.value.sync("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db_comments",
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
  storage.value.replicate.to("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db")
}


// Fetch data san alldocs, avec index sur likes:
       const fetchData = () => {
  if (!storage.value) return;
  storage.value.find({
    selector: {
      post_likes: { $gte: 0 }
    },
    sort: [{ post_likes: "desc" }],
    limit: 10,
    skip: page.value * 10
  })
    .then((result: any) => {
      console.log('=> Posts récupérés triés par likes :', result.docs);
      postsData.value = result.docs.filter(d => !(d._id.startsWith("_design")));
    })
    .catch((err: any) => {
      console.error('=> Erreur lors de la récupération des posts triés :', err);
    });

  // Gestion des commentaires
  if (!storageComments.value) return;
  storageComments.value.find({
    selector: {
      post_id: { $exists: true }
    },
  })
    .then((result: any) => {
      console.log('=> Commentaires récupérés :', result.docs);
      commentsData.value = result.docs.filter(c => !(c._id.startsWith("_design")));
    })
    .catch((err: any) => {
      console.error('=> Erreur lors de la récupération des commentaires :', err);
    });
}




const doc = {
  post_name: 'Super Document',
  post_content: 'SuperContenu',
  post_likes:0,
  attributes: {
    creation_date: new Date()
  }
}


const addDoc = (doc: any) => {
  storage.value
    .post(doc)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const addCom = (post_id: any) => {
   if (!storageComments.value) {
    console.warn("La collection comments n'existe pas");
    return;
  }
   const newCom = {
    post_id,
    comment_author: "Quelqu'un de très chouette sans doute",
    comment_content: needleComment.value || "Super commentaire",
    attributes: {
      creation_date: new Date()
    }
  };

  needleComment.value = "";

  storageComments.value.post(newCom).then(fetchData);
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
    .put(com)
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

// Ajouter une image
const selectImg = function (event: any, post: Post) {
  const file = event.target.files[0];
  if (!file) return;

  if (!file.type.startsWith('image/')) {
    alert('Veuillez sélectionner une image');
    return;
  }

  storage.value.putAttachment(post._id, file.name, post._rev, file, file.type)
    .then((result: any) => {
      console.log('Image attachée avec succès:', result);
      fetchData();
    })
    .catch((err: any) => {
      console.error('Erreur lors de l\'attachement:', err);
    });
}


// Supprimer une image
const deleteImg = function (post: Post, attachmentName: string) {
  storage.value.removeAttachment(post._id, attachmentName, post._rev)
    .then((result: any) => {
      console.log('Image supprimée avec succès:', result);
      fetchData();
    })
    .catch((err: any) => {
      console.error("Erreur suppression média :", err);
    });
};


// Afficher une image
const displayImage = (post: Post, attachName: string, imgElement: HTMLImageElement) => {
  storage.value.getAttachment(post._id, attachName)
    .then((blob: Blob) => {
      const url = URL.createObjectURL(blob);
      imgElement.src = url;
    })
    .catch((err: any) => {
      console.error("Erreur chargement image:", err);
    });
};




</script>

<template>
  <h1>Réseau social 100% legit, très fonctionnel, très élégant</h1>

  <label for="mode" name="mode">Mode offline</label>
  <input type="checkbox" id="mode" name="mode" v-model="offline" @change="handleChange">
<br>
  <button @click="addDoc(doc)">Nouveau Super Document</button>
  <button @click="creat100Docs()">Générer 100 documents</button>
  <br>
  <input type="text" placeholder="Search" @keyup.enter="search">
  <br>
  <p>  Nombre de post : {{ postsData.length }}</p>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <hr/>
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
    <p>{{ post.post_likes }}</p>

    <input type="text" name="needle" placeholder="Éditer le post" v-model="needle" @blur="updateDoc(post)">
    <br>
    <label for="file">Ajouter une image</label>
     <input type="file" name="img" accept="image/*" @change="selectImg($event, post)">

        <div v-if="post._attachments && Object.keys(post._attachments).length > 0">
          <h4>Image(s) :</h4>
          <div v-for="(attachment, name) in post._attachments" :key="name">
            <img :ref="(el) => el && displayImage(post, name, el)" />



            <button @click="deleteImg(post, name)">Supprimer l'image</button>
          </div>
        </div>
    <br>

    <button @click="deleteDoc(post._id,post._rev)">Supprimer Super Document</button>
    <br>
    <button @click="likePost(post)">J'aime!</button>
    <button @blur="addCom(post._id)">Commenter</button>

    <article v-for="comment in getComments(post._id)" v-bind:key="(comment as any).id">

          <ul>
            <li>
              <h2>{{ comment.comment_content }}</h2>

              <input type="text" name="needleComments" placeholder="Éditer le commentaire" v-model="needleComment" @blur="updateCom(comment)">
              <button @click="deleteCom(comment._id, comment._rev)">Supprimer le commentaire</button>
            </li>
          </ul>

        </article>

  </article>
  <button @click="page++; fetchData()">Voir les 10 suivants</button>

  <button @click="replicateDB()">Valider les changements</button>

</template>
