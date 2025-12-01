<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// Type des posts
interface Post {
  _id?: string
  post_name: string
  post_content: string
}

// Bases de données
const localDB = new PouchDB('infraddonn2_local')
const remoteDB = new PouchDB('http://admin:1234@localhost:5984/infraddonn2')

// Refs
const posts = ref<Post[]>([])
const formTitle = ref("")
const formContent = ref("")

// Charger les posts depuis la base locale
const loadPosts = async () => {
  try {
    const result = await localDB.allDocs({ include_docs: true })
    posts.value = result.rows.map(r => r.doc as Post)
  } catch (err) {
    console.error("Erreur loadPosts :", err)
  }
}

// Créer un post dans la base locale
const createPost = async () => {
  if (!formTitle.value || !formContent.value) return

  const newPost: Post = {
    _id: "post:" + Date.now(),
    post_name: formTitle.value,
    post_content: formContent.value
  }

  await localDB.put(newPost)
  await loadPosts()

  formTitle.value = ""
  formContent.value = ""
}

// Supprimer un post (locale → sync auto)
const deletePost = async (id: string) => {
  try {
    const doc = await localDB.get(id)
    await localDB.remove(doc)
    await loadPosts()
  } catch (err) {
    console.error("Erreur deletePost :", err)
  }
}

// Réplication locale ↔ distante (continue)
const startSync = () => {
  localDB.sync(remoteDB, {
    live: true,
    retry: true
  })
  .on('change', () => {
    console.log("Changement détecté → mise à jour de l'affichage")
    loadPosts()
  })
  .on('error', (err) => {
    console.error("Erreur de réplication :", err)
  })

  console.log("Réplication bidirectionnelle activée ✔")
}

onMounted(async () => {
  await loadPosts()
  startSync()
})
</script>

<template>
  <h1>Posts</h1>

  <h2>Créer un post</h2>

  <form
    @submit.prevent="createPost"
    style="display:flex; flex-direction:column; width:300px; gap:10px;"
  >
    <input v-model="formTitle" type="text" placeholder="Titre" required />
    <textarea v-model="formContent" placeholder="Contenu" rows="4" required></textarea>
    <button type="submit">Créer un post</button>
  </form>

  <article
    v-for="post in posts"
    :key="post._id"
    style="border:1px solid #ccc; padding:10px; margin:10px 0;"
  >
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>

    <button @click="deletePost(post._id!)" style="margin-top:10px;">
      Supprimer
    </button>
  </article>
</template>
