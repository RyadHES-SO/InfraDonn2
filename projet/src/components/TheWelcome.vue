<script setup lang="ts">
import { onMounted, ref, watch } from "vue"
import PouchDB from "pouchdb"
import PouchFind from "pouchdb-find"

PouchDB.plugin(PouchFind)

interface Post {
  _id?: string
  post_name: string
  post_content: string
}

const localDB = new PouchDB("infraddonn2_local")
const remoteDB = new PouchDB("http://admin:1234@localhost:5984/infraddonn2")

const posts = ref<Post[]>([])
const formTitle = ref("")
const formContent = ref("")
const search = ref("")

const loadPosts = async () => {
  const r = await localDB.allDocs({ include_docs: true })
  posts.value = r.rows.map(x => x.doc as Post)
}

const createPost = async () => {
  if (!formTitle.value || !formContent.value) return
  await localDB.put({
    _id: "post:" + Date.now(),
    post_name: formTitle.value,
    post_content: formContent.value
  })
  formTitle.value = ""
  formContent.value = ""
  loadPosts()
}

const deletePost = async (id: string) => {
  const doc = await localDB.get(id)
  await localDB.remove(doc)
  loadPosts()
}

localDB.createIndex({
  index: { fields: ["post_name", "post_content"] }
})

const searchPosts = async () => {
  if (!search.value) return loadPosts()
  const r = await localDB.find({
    selector: {
      $or: [
        { post_name: { $regex: search.value } },
        { post_content: { $regex: search.value } }
      ]
    }
  })
  posts.value = r.docs as Post[]
}

watch(search, searchPosts)

const startSync = () => {
  localDB
    .sync(remoteDB, { live: true, retry: true })
    .on("change", loadPosts)
}

onMounted(() => {
  loadPosts()
  startSync()
})
</script>

<template>
  <h1>Posts</h1>

  <input v-model="search" type="text" placeholder="Recherche..." style="margin-bottom:20px; width:300px;" />

  <h2>Créer un post</h2>

  <form @submit.prevent="createPost" style="display:flex; flex-direction:column; width:300px; gap:10px;">
    <input v-model="formTitle" type="text" placeholder="Titre" />
    <textarea v-model="formContent" placeholder="Contenu" rows="4"></textarea>
    <button type="submit">Créer un post</button>
  </form>

  <article v-for="p in posts" :key="p._id" style="border:1px solid #ccc; padding:10px; margin:10px 0;">
    <h2>{{ p.post_name }}</h2>
    <p>{{ p.post_content }}</p>
    <button @click="deletePost(p._id!)">Supprimer</button>
  </article>
</template>
