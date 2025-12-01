<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb'

declare interface Post {
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
}

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:1234@localhost:5984/infraddonn2')
  if (db) {
    console.log("Connecté à la collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

//base local¨
const localDB = new PouchDB('infraddonn2_local')

const syncDatabases = async () => {
  if (!storage.value) return

  const remoteDB = new PouchDB('http://admin:1234@localhost:5984/infraddonn2')

  try {
    await localDB.sync(remoteDB, {
      live: false,      // false = synchronisation ponctuelle
      retry: true       // retry si problème de connexion
    })
    console.log("Synchronisation terminée")
    await fetchData() // Recharge la vue après la synchro
  } catch (error) {
    console.error("Erreur lors de la synchronisation :", error)
  }
}


// Récupération des données
const fetchData = async () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    postsData.value = result.rows
      .filter(row => row.doc)
      .map(row => row.doc as Post)
    console.log('Données récupérées :', postsData.value)
  } catch (error) {
    console.error('Erreur lors de la récupération des données :', error)
  }
}

// CREATE
const createPost = async () => {
  if (!storage.value) return

  try {
    await storage.value.get('14852340') // si existe
    console.log("Le document existe déjà")
  } catch (err: any) {
    if (err.status === 404) {
      // Si existe pas
      await storage.value.put({
        _id: '14852340',
        post_name: "PostCreeavecPUT",
        post_content: "LECONTENT ECRIT"
      })
    } else {
      console.error("Erreur inattendue :", err)
    }
  }

  await fetchData()
}

//UPDATE
const updatePost = async () => {
  if (!storage.value) return

  try {
    const doc = await storage.value.get('14852340')
    doc.post_content = "Contenu modifié"
    doc.post_name = "Titre modifié"
    await storage.value.put(doc)

    await fetchData()
  } catch (error) {
    console.error("Erreur en mettant à jour :", error)
  }
}

//DELETE
const deletePost = async () => {
  if (!storage.value) return

  try {
    const doc = await storage.value.get('14852340')
    await storage.value.remove(doc)
    await fetchData()
    console.log("Document supprimé")
  } catch (error) {
    console.error("Erreur en supprimant :", error)
  }
}


onMounted(async () => {
  console.log('=> Composant initialisé');
  initDatabase()

  await createPost()
  await updatePost()
  await deletePost()
  await fetchData()
});

</script>

<template>
  <h1>Fetch Data</h1>
  <button @click="createPost">Créer un post démo</button>
  <button @click="syncDatabases">Synchroniser avec la base distante</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
</template>
