<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

// Interfaces
interface Post {
  nom: string
  age: number
  ville: string
  _id?: string
  _rev?: string
}

interface CommentDoc {
  postId: string
  texte: string
  auteur: string
  _id?: string
  _rev?: string
}

// Databases
const postsDB = ref<PouchDB.Database<Post> | null>(null)
const postsData = ref<Post[]>([])
let postsChanges: any = null
let postsSync: any = null

const commentsDB = ref<PouchDB.Database<CommentDoc> | null>(null)
const commentsData = ref<CommentDoc[]>([])
let commentsChanges: any = null
let commentsSync: any = null

// Mode offline
const isOffline = ref(false)

// Init
const initDatabases = () => {
  postsDB.value = new PouchDB<Post>('posts')
  commentsDB.value = new PouchDB<CommentDoc>('comments')

  console.log('Connecté DB Posts :', postsDB.value.name)
  console.log('Connecté DB Comments :', commentsDB.value.name)
}

// Indexs
const createIndexes = async () => {
  await postsDB.value?.createIndex({ index: { fields: ['nom'] } })
  await commentsDB.value?.createIndex({ index: { fields: ['postId'] } })
}

// Fetch
const fetchPosts = () => {
  postsDB.value
    ?.allDocs({ include_docs: true })
    .then((result) => (postsData.value = result.rows.map((r) => r.doc!)))
}

const fetchComments = () => {
  commentsDB.value
    ?.allDocs({ include_docs: true })
    .then((result) => (commentsData.value = result.rows.map((r) => r.doc!)))
}

// Watchers
const startWatchers = () => {
  postsChanges?.cancel()
  postsChanges = postsDB.value
    ?.changes({
      live: true,
      include_docs: true,
      since: 'now',
    })
    .on('change', fetchPosts)

  commentsChanges?.cancel()
  commentsChanges = commentsDB.value
    ?.changes({
      live: true,
      include_docs: true,
      since: 'now',
    })
    .on('change', fetchComments)
}

// Synchro
const remotePosts = 'http://admin:cestcatastrophiiique@localhost:5984/posts'
const remoteComments = 'http://admin:cestcatastrophiiique@localhost:5984/comments'

const startSync = () => {
  postsSync?.cancel()
  commentsSync?.cancel()

  postsSync = postsDB.value?.sync(remotePosts, { live: true, retry: true })
  commentsSync = commentsDB.value?.sync(remoteComments, { live: true, retry: true })

  console.log('Synchronisation activée')
}

const stopSync = () => {
  postsSync?.cancel()
  commentsSync?.cancel()

  postsSync = null
  commentsSync = null

  console.log('Synchronisation arrêtée')
}

const toggleOffline = () => {
  isOffline.value = !isOffline.value
  isOffline.value ? stopSync() : startSync()
}

// CRUD Posts
const createPost = (doc: Post) => postsDB.value?.post(doc).then(fetchPosts)
const deletePost = (doc: Post) => postsDB.value?.remove(doc).then(fetchPosts)
const updatePost = (doc: Post) => {
  const updatedPost = { ...doc, ville: 'Lausanne' }
  postsDB.value?.put(updatedPost).then(fetchPosts)
}

// CRUD Comments
const createComment = (comment: CommentDoc) => commentsDB.value?.post(comment).then(fetchComments)

const deleteComment = (comment: CommentDoc) => commentsDB.value?.remove(comment).then(fetchComments)

// Recherche
const searchPost = ref('')

const searchPostsDB = async () => {
  if (!searchPost.value.trim()) return fetchPosts()
  const res = await postsDB.value?.find({
    selector: { nom: { $regex: RegExp(searchPost.value, 'i') } },
  })
  postsData.value = res?.docs ?? []
}

// Mount
onMounted(() => {
  initDatabases()
  createIndexes()
  fetchPosts()
  fetchComments()
  startWatchers()
  startSync()
})
</script>

<template>
  <button @click="toggleOffline">
    {{ isOffline ? 'Passer online' : 'Passer offline' }}
  </button>

  <h2>Posts</h2>

  <input v-model="searchPost" @input="searchPostsDB" placeholder="Rechercher un post..." />

  <ul>
    <li v-for="p in postsData" :key="p._id">
      {{ p.nom }} – {{ p.ville }}
      <button @click="updatePost(p)">Modifier</button>
      <button @click="deletePost(p)">Supprimer</button>

      <h4>Commentaires :</h4>
      <ul>
        <li v-for="c in commentsData.filter((cm) => cm.postId === p._id)" :key="c._id">
          {{ c.texte }} — {{ c.auteur }}
          <button @click="deleteComment(c)">Supprimer</button>
        </li>
      </ul>

      <button
        @click="createComment({ postId: p._id!, texte: 'Nouveau comment', auteur: 'Mowgli-san' })"
      >
        Ajouter commentaire
      </button>
    </li>
  </ul>

  <button @click="createPost({ nom: 'Marie', age: 22, ville: 'Tokyo' })">Créer Post</button>
</template>
