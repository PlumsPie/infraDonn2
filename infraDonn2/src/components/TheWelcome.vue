<script setup lang="ts">
// npm run dev to restart server
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
  // J'ai ajouté ces deux attributs pour faciliter l'affichage des commentaires,
  // en ayant toujours le dernier à portée de main, et en ayant la possibilité de stocker
  // les autres le temps de l'affichage, mais vu que ce n'est pas par défaut ça évite quand
  // même de tout charger pour rien, seulement si l'utilisateur le demande
  lastComment?: Comment | null
  allComments?: Comment[]
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
const needle = ref('');
const needleComment = ref('');
const newCommentText = ref('');

// Données stockées
const postsData = ref<Post[]>([])
const showAllComments = ref<Record<string, boolean>>({})
const page = ref(0);

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()

})


// Initialisation de la base de données

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

  // Création des index
  // Index sur post_likes pour trier les posts

    storage.value.createIndex({
      index: {
        fields: ['post_likes']
      }
    }).then(console.log("L'index sur post_like a été créé avec succès"));

    // Index sur post_id et creation_date pour récupérer les derniers commentaires
  storageComments.value.createIndex({
    index: {
      fields: ['post_id', 'attributes.creation_date']
    }
  }).then(() => console.log("Les index sur post_id et creation_date ont été créé avec succès"));


    fetchData()
    if (!offline.value) {
      startSync();
    }
}

const startSync = () => {
  if (sync.value) {
    console.log("La synchronisation est déjà en cours");
    return;
  }
  if (!storage.value) {
    console.warn('Pas de base de données locale à synchroniser');
    return;
  }
  if (syncComments.value) {
    console.log("La synchronisation des commentaires est déjà en cours");
    return;
  }
  if (!storageComments.value) {
    console.warn('Pas de base de données locale à synchroniser');
    return;
  }
// synchronisation des posts
  sync.value = storage.value.sync("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db",
    {
      live: true,
      retry: true
    })
    .on('change', (info: any) => {
      console.log("Sync change posts", info);
      fetchData();
    })

// synchronisation des commentaires
    syncComments.value = storageComments.value.sync("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db_comments",
    {
      live: true,
      retry: true
    })
    .on('change', (info: any) => {
      console.log("Sync change commentaires", info);
      fetchData();
    })
  console.log('Sync started');
}


const stopSync = () => {
  try {
    if (sync.value){
    sync.value.cancel()
    sync.value = null
    }
    if(syncComments.value){
    syncComments.value.cancel()
    syncComments.value = null
    }
    console.log('Synchronisation annulée')
  } catch (err) {
    console.error('Erreur lors de l\'arrêt de la synchronisation', err)
  }
}


const handleChange = () => {

  if (!offline.value) {
    console.log("Mode ONLINE => lancement de la synchronisation");
    startSync();
  } else {
    console.log("Mode OFFLINE → arrêt de la synchronisation");
    stopSync();
  }
}

const replicateDB = ()=>{
  storage.value.replicate.to("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db")
  storageComments.value.replicate.to("http://admin:ThDR.Tk9P_q2gKkdABhB@localhost:5984/sarah_furrer_infradonn_db_comments");

}


// Fetch data sans alldocs, avec index sur likes:
  const fetchData = async () => {
  if (!storage.value) return;

  try {
    // Récupération des 10 posts les plus likés avec pagination
    const result = await storage.value.find({
      selector: {
        post_likes: { $gte: 0 }
      },
      sort: [{ post_likes: "desc" }],
      limit: 10,
      skip: page.value * 10
    });

    console.log('=> 10 Posts récupérés triés par likes :', result.docs);
    // pour ne pas inclure d'index par accident:
    const posts = result.docs.filter((d: any) => !d._id.startsWith("_design"));

    // Pour chaque post, on récupère le dernier commentaire
    for (const post of posts) {
      post.lastComment = await getLastComment(post._id);
    }

    postsData.value = posts;

  } catch (err) {
    console.error('=> Erreur lors de la récupération des posts :', err);
  }
}

// Récupérer le dernier commentaire d'un post
const getLastComment = async (postId: string): Promise<Comment | null> => {
  if (!storageComments.value) return null;

  try {
    const result = await storageComments.value.find({
      selector: {
        post_id: postId,
        'attributes.creation_date': { $exists: true }
      },
      sort: [{ 'attributes.creation_date': 'desc' }],
      limit: 1
    });

    return result.docs.length > 0 ? result.docs[0] : null;
  } catch (err) {
    console.error('Erreur récupération dernier commentaire:', err);
    return null;
  }
}

// Récupérer tous les commentaires d'un post
const handleComments = async (post: Post) => {
  const postId = post._id;

  // Si déjà affiché, masquer les commentaires
  if (showAllComments.value[postId]) {
    showAllComments.value[postId] = false;
    post.allComments = undefined;
    return;
  }

  // Sinon, on charger depuis la base de données
  if (!storageComments.value) return;

  try {
    const result = await storageComments.value.find({
      selector: {
        post_id: postId,
        'attributes.creation_date': { $exists: true }
      },
      sort: [{ 'attributes.creation_date': 'desc' }]
    });

    post.allComments = result.docs.filter((c: any) => !c._id.startsWith("_design"));
    showAllComments.value[postId] = true;
    console.log(`${post.allComments.length} commentaires chargés pour ${postId}`);
  } catch (err) {
    console.error('Erreur chargement commentaires:', err);
  }
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
const addCom = (post: Post) => {
   if (!storageComments.value) {
    console.warn("La collection comments n'existe pas");
    return;
  }
  storageComments.value.post({
    post_id: post._id,
    comment_author: "Quelqu'un de très chouette sans doute",
    comment_content: newCommentText.value,
    attributes: { creation_date: new Date() }
  }).then(() => {
    newCommentText.value = "";
    fetchData();
    // Si tous les commentaires sont affichés, les recharger
    if (showAllComments.value[post._id]) {
      handleComments(post).then(() => {
        showAllComments.value[post._id] = false;
        handleComments(post);
      });
    }
  })
}
const updateDoc = (doc: any) => {
  if (!needle.value) return;
  doc.post_name= needle.value;
  needle.value = '';
  storage.value
    .put(doc)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
    })
}
const updateCom = (com:any, post:Post) => {
    if (!needleComment.value) return;
  com.comment_content= needleComment.value;
  needleComment.value = '';
  storageComments.value
    .put(com)
    .then(() => fetchData())
    .catch(function (err: any) {
      console.log(err)
      // Recharger les commentaires si affichés
      if (showAllComments.value[post._id]) {
        handleComments(post).then(() => {
          showAllComments.value[post._id] = false;
          handleComments(post);
        });
      }
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
const deleteCom = (comId: any, comRev: any, post:Post) => {

  storageComments.value
    .remove(comId, comRev)
    .then(() => {fetchData()
  if (showAllComments.value[post._id]) {
        handleComments(post).then(() => {
          showAllComments.value[post._id] = false;
          handleComments(post);
        });
      }
      })
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
      post_content: words[Math.floor(Math.random() * words.length)],
      post_likes: Math.floor(Math.random() * 10),  // Ajouté
  attributes: { creation_date: new Date() }
    });
  }

  fetchData();
};


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
// Gestion de la pagination
const nextPage = () => {
  page.value++;
  fetchData();
}

const prevPage = () => {
  if (page.value > 0) {
    page.value--;
    fetchData();
  }
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
      console.error("Erreur lors de la suppression de l'image :", err);
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
  <button @click="addDoc(doc)">Nouveau Super Post</button>
  <button @click="creat100Docs()">Générer 100 Posts</button>
  <br>
  <input type="text" placeholder="Search" @keyup.enter="search">
  <br>
  <p>  Nombre de post : {{ postsData.length }} Page {{ page + 1 }}</p>

<!-- Pagination -->
  <button @click="prevPage()" :disabled="page === 0">Page précédente</button>
  <button @click="nextPage()">Page suivante</button>

<!-- Affichage posts -->
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <hr/>
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
    <p>{{ post.post_likes }}</p>

    <input type="text" name="needle" placeholder="Éditer le post" v-model="needle" @blur="updateDoc(post)">
    <br>

    <!-- Gestion des image -->
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

    <button @click="deleteDoc(post._id,post._rev)">Supprimer Super Post</button>
    <br>
    <button @click="likePost(post)">J'aime!</button>


    <!-- Gestion des commentaires -->

     <div v-if="post.lastComment">
        <p>Dernier commentaire : {{ post.lastComment.comment_content }}</p>
      </div>
      <p v-else>Aucun commentaire pour le moment</p>


      <button @click="handleComments(post)">
        {{ showAllComments[post._id] ? 'Masquer' : 'Voir tous' }}
      </button>

    <!-- Tous les commentaires -->
      <div v-if="showAllComments[post._id] && post.allComments">
        <h5>Tous les commentaires :</h5>
        <article v-for="comment in post.allComments" :key="comment._id"
                 style="margin: 10px 0; padding: 10px; background: #fff; border-left: 3px solid #007bff;">
          <p>{{ comment.comment_content }}</p>
          <small>par {{ comment.comment_author }}</small>
          <br>
          <input
            type="text"
            placeholder="Éditer"
            v-model="needleComment"
            @blur="updateCom(comment, post)"
          >
          <button @click="deleteCom(comment._id, comment._rev, post)">Supprimer</button>
        </article>
      </div>
    <input
          type="text"
          v-model="newCommentText"
          placeholder="Écrire un commentaire..."
          @keyup.enter="addCom(post)"
        >
        <button @click="addCom(post)">Commenter</button>
  </article>

<button @click="replicateDB()">Valider les changements</button>
</template>
