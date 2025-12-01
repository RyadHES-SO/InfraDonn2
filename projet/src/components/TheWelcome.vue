<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// Type de données
interface Post {
  _id?: string
  post_name: string
  post_content: string
}

// Références
const storage = ref<any>(null)
const posts = ref<Post[]>([])
const title = ref("")
const content = ref("")

// Base locale et distante
const localDB = new PouchDB('infraddonn2_local')
const remoteURL = 'http://admin:1234@localhost:5984/infraddonn2'

// Connexion à la base distante
const initDB = () => {
  console.log("Connexion à la base distante...")
  storage.value = new PouchDB(remoteURL)
}

// Récupération des posts
const loadPosts = async () => {
  if (!storage.value) return

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    posts.value = result.rows.map(r => r.doc as Post)
    console.log("Posts chargés :", posts.value)
  } catch (err) {
    console.error("Erreur loadPosts :", err)
  }
}

// Création d'un post
const createPost = async () => {
  if (!title.value || !content.value || !storage.value) return

  const newPost: Post = {
    _id: "post:" + Date.now(),
    post_name: title.value,
    post_content: content.value
  }

  await storage.value.put(newPost)
  await loadPosts()

  title.value = ""
  content.value = ""
}

// Suppression d'un post (réutilisable)
const removePost = async (id: string) => {
  if (!storage.value) return

  try {
    const doc = await storage.value.get(id)
    await storage.value.remove(doc)
    await loadPosts()
  } catch (err) {
    console.error("Erreur removePost :", err)
  }
}

// Synchronisation local <-> distant
const syncDB = async () => {
  try {
    const remoteDB = new PouchDB(remoteURL)
    await localDB.sync(remoteDB, { live: false, retry: true })
    console.log("Synchronisation OK")
    await loadPosts()
  } catch (err) {
    console.error("Erreur syncDB :", err)
  }
}

onMounted(async () => {
  initDB()
  await loadPosts()
})
</script>

<template>
  <h1>Posts</h1>

  <h2>Créer un post</h2>

  <form 
    @submit.prevent="createPost"
    style="display:flex; flex-direction:column; width:300px; gap:10px;"
  >
    <input v-model="title" type="text" placeholder="Titre" required />
    <textarea v-model="content" placeholder="Contenu" rows="4" required></textarea>
    <button type="submit">Créer un post</button>
  </form>

  <button @click="syncDB" style="margin-top:20px;">
    Synchroniser
  </button>

  <article
    v-for="post in posts"
    :key="post._id"
    style="border:1px solid #ccc; padding:10px; margin:10px 0;"
  >
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>

    <button @click="removePost(post._id!)" style="margin-top:10px;">
      Supprimer
    </button>
  </article>
</template>
