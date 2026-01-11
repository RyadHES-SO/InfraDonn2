<script setup lang="ts">
import { onMounted, ref } from "vue"
import PouchDB from "pouchdb"
import PouchFind from "pouchdb-find"

PouchDB.plugin(PouchFind)

interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  likes: number
  _attachments?: Record<string, { content_type: string }>
  imageUrl?: string
}

interface Comment {
  _id?: string
  _rev?: string
  comment_text: string
  post_id: string
}

const postsDB = new PouchDB("posts_ait_slimane_ryad")
const commentsDB = new PouchDB("comments_ait_slimane_ryad")
const remotePostsDB = new PouchDB("http://admin:1234@localhost:5984/posts_ait_slimane_ryad")
const remoteCommentsDB = new PouchDB("http://admin:1234@localhost:5984/comments_ait_slimane_ryad")

const posts = ref<Post[]>([])
const comments = ref<Comment[]>([])

const postTitle = ref("")
const postContent = ref("")
const postMedia = ref<File | null>(null)
const editingPostId = ref<string | null>(null)

const commentTexts = ref<Record<string, string>>({})
const editingCommentId = ref<string | null>(null)
const showAllComments = ref<Record<string, boolean>>({})

const searchQuery = ref("")
const skip = ref(0)
const limit = 10
const sortedByLikes = ref(false)

const handleFileChange = (e: Event) => {
  const target = e.target as HTMLInputElement
  postMedia.value = target?.files?.[0] || null
}

const loadPosts = async () => {
  const r = await postsDB.allDocs({ include_docs: true, attachments: true })
  const docs = r.rows.map(r => r.doc as Post)
  for (const doc of docs) {
    if (doc._attachments) {
      const attachmentName = Object.keys(doc._attachments)[0]
      const blob = await postsDB.getAttachment(doc._id!, attachmentName)
      doc.imageUrl = URL.createObjectURL(blob)
    } else {
      doc.imageUrl = undefined
    }
  }

  // Trier selon le mode
  posts.value = docs
    .sort((a, b) =>
      sortedByLikes.value ? b.likes - a.likes : b._id!.localeCompare(a._id!)
    )
    .slice(skip.value, skip.value + limit)
}

const loadComments = async () => {
  const r = await commentsDB.allDocs({ include_docs: true })
  comments.value = r.rows.map(r => r.doc as Comment)
}

const commentsForPost = (postId: string) => comments.value.filter(c => c.post_id === postId)

const lastCommentForPost = (postId: string) => {
  const list = commentsForPost(postId)
  return list.length ? list[list.length - 1] : null
}

const createPost = async () => {
  if (!postTitle.value || !postContent.value) return

  const id = editingPostId.value || "post:" + Date.now()
  const doc: Post = {
    _id: id,
    post_name: postTitle.value,
    post_content: postContent.value,
    likes: editingPostId.value ? (await postsDB.get(id)).likes : 0
  }
  if (editingPostId.value) {
    doc._rev = (await postsDB.get(id))._rev
  }
  await postsDB.put(doc)

  if (postMedia.value) {
    const rev = (await postsDB.get(id, { attachments: true }))._rev
    await postsDB.putAttachment(id, postMedia.value.name, rev, postMedia.value, postMedia.value.type)
  }

  postTitle.value = ""
  postContent.value = ""
  postMedia.value = null
  editingPostId.value = null
  loadPosts()
}

const editPost = (p: Post) => {
  postTitle.value = p.post_name
  postContent.value = p.post_content
  editingPostId.value = p._id || null
}

const deletePost = async (id: string) => {
  const doc = await postsDB.get(id)
  await postsDB.remove(doc)
  loadPosts()
}

const deleteMedia = async (post: Post) => {
  if (!post._attachments) return
  const attachmentName = Object.keys(post._attachments)[0]
  const doc = await postsDB.get(post._id!, { attachments: true })
  await postsDB.removeAttachment(post._id!, attachmentName, doc._rev!)
  loadPosts()
}

const likePost = async (post: Post) => {
  const doc = await postsDB.get(post._id!)
  doc.likes = (doc.likes || 0) + 1
  await postsDB.put(doc)
  loadPosts()
}

const createComment = async (postId: string) => {
  const text = commentTexts.value[postId]
  if (!text) return

  if (editingCommentId.value) {
    const doc = await commentsDB.get(editingCommentId.value)
    doc.comment_text = text
    await commentsDB.put(doc)
    editingCommentId.value = null
  } else {
    await commentsDB.put({
      _id: "comment:" + Date.now(),
      comment_text: text,
      post_id: postId
    })
  }

  commentTexts.value[postId] = ""
  loadComments()
}

const editComment = async (c: Comment) => {
  commentTexts.value[c.post_id] = c.comment_text
  editingCommentId.value = c._id || null
}

const deleteComment = async (c: Comment) => {
  const doc = await commentsDB.get(c._id!)
  await commentsDB.remove(doc)
  loadComments()
}

const searchPosts = async () => {
  const q = searchQuery.value.trim()
  if (!q) return loadPosts()
  posts.value = (await postsDB.find({ selector: { post_name: { $regex: q } } })).docs as Post[]
}

const sortToggle = () => {
  sortedByLikes.value = !sortedByLikes.value
  loadPosts()
}

const nextPosts = () => {
  skip.value += limit
  loadPosts()
}

const prevPosts = () => {
  skip.value = Math.max(0, skip.value - limit)
  loadPosts()
}

const startSync = () => {
  postsDB.sync(remotePostsDB, { live: true, retry: true })
  commentsDB.sync(remoteCommentsDB, { live: true, retry: true })
}

onMounted(() => {
  loadPosts()
  loadComments()
  startSync()
})
</script>

<template>
  <h1>Posts</h1>

  <input v-model="searchQuery" placeholder="Rechercher..." @input="searchPosts"
    style="width:300px; margin-bottom:10px;" />

  <h2>Créer / Modifier un post</h2>
  <form @submit.prevent="createPost" style="display:flex; flex-direction:column; gap:10px; width:300px;">
    <input v-model="postTitle" placeholder="Titre" />
    <textarea v-model="postContent" placeholder="Contenu" rows="4"></textarea>
    <input type="file" @change="handleFileChange" />
    <button type="submit">{{ editingPostId ? "Modifier" : "Créer" }}</button>
  </form>

  <button @click="sortToggle" style="margin-top:10px;">
    {{ sortedByLikes ? "Réinitialiser l'ordre" : "Trier par likes" }}
  </button>

  <article v-for="p in posts" :key="p._id" style="border:1px solid #ccc; padding:10px; margin:20px 0;">
    <h2>{{ p.post_name }}</h2>
    <p>{{ p.post_content }}</p>
    <p>❤️ {{ p.likes }}</p>

    <div v-if="p.imageUrl">
      <img :src="p.imageUrl" style="max-width:200px; display:block; margin:10px 0;" />
      <button @click="deleteMedia(p)">Supprimer le média</button>
    </div>

    <button @click="likePost(p)">👍 Liker</button>
    <button @click="editPost(p)">Modifier</button>
    <button @click="deletePost(p._id!)">Supprimer</button>

    <div style="margin-top:10px;">
      <h3>Commentaires</h3>
      <p v-if="lastCommentForPost(p._id!)">
        Dernier : {{ lastCommentForPost(p._id!)?.comment_text }}
      </p>
      <button @click="showAllComments[p._id!] = !showAllComments[p._id!]">
        {{ showAllComments[p._id!] ? "Cacher tous les commentaires" : "Voir tous les commentaires" }}
      </button>

      <article v-for="c in showAllComments[p._id!] ? commentsForPost(p._id!) : []" :key="c._id"
        style="border:1px solid #eee; margin:5px 0; padding:5px;">
        {{ c.comment_text }}
        <button @click="editComment(c)">Modifier</button>
        <button @click="deleteComment(c)">Supprimer</button>
      </article>

      <textarea v-model="commentTexts[p._id!]" placeholder="Ajouter un commentaire" rows="2"
        style="width:100%; margin-top:5px;"></textarea>
      <button @click="() => createComment(p._id!)" style="margin-top:5px;">Ajouter</button>
    </div>
  </article>

  <div style="margin-top:20px;">
    <button @click="prevPosts" :disabled="skip === 0">Précédents</button>
    <button @click="nextPosts">Suivants</button>
  </div>
</template>

/*
Pour l’instant, j’ai mis en place une réplication live entre la base locale et la base distante pour que les posts et
commentaires soient toujours synchronisés. J’utilise allDocs({ include_docs: true, attachments: true }) pour récupérer
les documents et leurs images, ce qui est suffisant pour un nombre limité de posts et commentaires. Pour optimiser,
surtout si la base devient volumineuse, il serait plus efficace d’utiliser des index avec find() pour ne récupérer que
les documents nécessaires, de charger les pièces jointes uniquement à la demande et de filtrer la réplication pour
limiter la quantité de données échangées. Ces choix permettent de garder le code simple et fonctionnel tout en restant
attentif à la performance.
*/