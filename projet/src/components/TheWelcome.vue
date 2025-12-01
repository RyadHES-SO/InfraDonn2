<script setup lang="ts">
import { onMounted, ref, watch } from "vue"
import PouchDB from "pouchdb"
import PouchFind from "pouchdb-find"

PouchDB.plugin(PouchFind)

interface Post { _id?: string; _rev?: string; post_name: string; post_content: string }
interface Comment { _id?: string; _rev?: string; comment_text: string; post_id: string }

const localDB = new PouchDB("infraddonn2_local")
const remoteDB = new PouchDB("http://admin:1234@localhost:5984/infraddonn2")

const posts = ref<Post[]>([])
const postTitle = ref("")
const postContent = ref("")
const editingPostId = ref<string | null>(null)

const comments = ref<Comment[]>([])
const commentText = ref("")
const commentPostId = ref<string | null>(null)
const editingCommentId = ref<string | null>(null)

const searchQuery = ref("")

const loadPosts = async () => {
  const r = await localDB.allDocs({ include_docs: true })
  posts.value = r.rows.filter(r => r.doc?.post_name).map(r => r.doc as Post)
}

const loadComments = async () => {
  const r = await localDB.allDocs({ include_docs: true })
  comments.value = r.rows.filter(r => r.doc?.comment_text).map(r => r.doc as Comment)
}

const createPost = async () => {
  if (!postTitle.value || !postContent.value) return
  if (editingPostId.value) { await updatePost(editingPostId.value); return }
  await localDB.put({ _id: "post:" + Date.now(), post_name: postTitle.value, post_content: postContent.value })
  postTitle.value = ""; postContent.value = ""; editingPostId.value = null; loadPosts()
}

const updatePost = async (id: string) => {
  const doc = await localDB.get(id)
  doc.post_name = postTitle.value; doc.post_content = postContent.value
  await localDB.put(doc)
  editingPostId.value = null; postTitle.value = ""; postContent.value = ""; loadPosts()
}

const editPost = (p: Post) => { postTitle.value = p.post_name; postContent.value = p.post_content; editingPostId.value = p._id || null }
const deletePost = async (id: string) => { const doc = await localDB.get(id); await localDB.remove(doc); loadPosts() }

const createComment = async () => {
  if (!commentText.value || !commentPostId.value) return
  if (editingCommentId.value) { await updateComment(editingCommentId.value); return }
  await localDB.put({ _id: "comment:" + Date.now(), comment_text: commentText.value, post_id: commentPostId.value })
  commentText.value = ""; commentPostId.value = null; editingCommentId.value = null; loadComments()
}

const updateComment = async (id: string) => {
  const doc = await localDB.get(id)
  doc.comment_text = commentText.value; doc.post_id = commentPostId.value!
  await localDB.put(doc)
  editingCommentId.value = null; commentText.value = ""; commentPostId.value = null; loadComments()
}

const editComment = (c: Comment) => { commentText.value = c.comment_text; commentPostId.value = c.post_id; editingCommentId.value = c._id || null }
const deleteComment = async (id: string) => { const doc = await localDB.get(id); await localDB.remove(doc); loadComments() }

const searchAll = async () => {
  const q = searchQuery.value.trim()
  if (!q) { loadPosts(); loadComments(); return }
  const postsRes = await localDB.find({ selector: { post_name: { $regex: q } } })
  posts.value = postsRes.docs as Post[]
  const commentsRes = await localDB.find({ selector: { comment_text: { $regex: q } } })
  comments.value = commentsRes.docs as Comment[]
}

watch(searchQuery, searchAll)
localDB.createIndex({ index: { fields: ["post_name", "post_content"] } })
localDB.createIndex({ index: { fields: ["comment_text", "post_id"] } })

const startSync = () => { localDB.sync(remoteDB, { live: true, retry: true }).on("change", () => { loadPosts(); loadComments() }) }
const manualSync = async () => { await localDB.sync(remoteDB, { live: false, retry: true }); loadPosts(); loadComments() }

onMounted(() => { loadPosts(); loadComments(); startSync() })
</script>

<template>
  <h1>Recherche globale</h1>
  <input v-model="searchQuery" placeholder="Rechercher posts et commentaires"
    style="width:300px; margin-bottom:20px;" />

  <h2>Créer / Modifier un post</h2>
  <form @submit.prevent="createPost" style="display:flex; flex-direction:column; width:300px; gap:10px;">
    <input v-model="postTitle" placeholder="Titre" />
    <textarea v-model="postContent" placeholder="Contenu" rows="4"></textarea>
    <button type="submit">{{ editingPostId ? "Modifier le post" : "Créer un post" }}</button>
  </form>

  <button @click="manualSync" style="margin:20px 0;">Synchroniser manuellement</button>

  <article v-for="p in posts" :key="p._id" style="border:1px solid #ccc; padding:10px; margin:10px 0;">
    <h2>{{ p.post_name }}</h2>
    <p>{{ p.post_content }}</p>
    <button @click="editPost(p)" style="margin-right:10px;">Modifier</button>
    <button @click="deletePost(p._id!)">Supprimer</button>
  </article>

  <h2>Commentaires</h2>
  <form @submit.prevent="createComment" style="display:flex; flex-direction:column; width:300px; gap:10px;">
    <textarea v-model="commentText" placeholder="Commentaire" rows="2"></textarea>
    <select v-model="commentPostId">
      <option disabled value="">Sélectionner un post</option>
      <option v-for="p in posts" :key="p._id" :value="p._id">{{ p.post_name }}</option>
    </select>
    <button type="submit">{{ editingCommentId ? "Modifier le commentaire" : "Créer un commentaire" }}</button>
  </form>

  <article v-for="c in comments" :key="c._id" style="border:1px solid #ccc; padding:10px; margin:10px 0;">
    <p>{{ c.comment_text }} (Post: {{posts.find(p => p._id === c.post_id)?.post_name || 'N/A'}})</p>
    <button @click="editComment(c)" style="margin-right:10px;">Modifier</button>
    <button @click="deleteComment(c._id!)">Supprimer</button>
  </article>
</template>
