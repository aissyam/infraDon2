<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

// Compteur
const counter = ref(0)
const increment = () => counter.value++

// Interfaces
declare interface CommentDoc {
  _id?: string
  _rev?: string
  postId: string
  text: string
  date: string
}

declare interface Post {
  _id?: string
  _rev?: string
  nom: string
  age: number
  ville: string
  sport: string
  likes: number
  displayComments?: CommentDoc[]
}

// Databases & State
const postsDB = ref<PouchDB.Database<Post> | null>(null)
const commentsDB = ref<PouchDB.Database<CommentDoc> | null>(null)
const postsData = ref<Post[]>([])
const isOffline = ref(false)
const searchTerm = ref('')

let postSyncHandler: PouchDB.Replication.Sync<any> | null = null
let commentSyncHandler: PouchDB.Replication.Sync<any> | null = null

// CouchDB URLs
const COUCH_URL_POSTS = 'http://admin:cestcatastrophiiique@localhost:5984/ma_collection'
const COUCH_URL_COMMENTS = 'http://admin:cestcatastrophiiique@localhost:5984/second'

// Init & Sync
const initDatabases = () => {
  console.log('=> Initialisation des 2 collections')
  postsDB.value = new PouchDB<Post>('Posts')
  commentsDB.value = new PouchDB<CommentDoc>('Comments')
  if (!isOffline.value) startLiveReplication()
}

const startLiveReplication = () => {
  if (!postsDB.value || !commentsDB.value || postSyncHandler) return

  postSyncHandler = postsDB.value
    .sync(COUCH_URL_POSTS, { live: true, retry: true })
    .on('change', fetchData)

  commentSyncHandler = commentsDB.value
    .sync(COUCH_URL_COMMENTS, { live: true, retry: true })
    .on('change', fetchData)
}

const toggleOfflineMode = () => {
  isOffline.value = !isOffline.value
  if (isOffline.value) {
    postSyncHandler?.cancel()
    commentSyncHandler?.cancel()
    postSyncHandler = null
    commentSyncHandler = null
    console.log('Mode Offline')
  } else {
    startLiveReplication()
    console.log('Mode Online')
  }
}

const MAJserveur = async () => {
  try {
    await postsDB.value?.replicate.to(COUCH_URL_POSTS)
    await postsDB.value?.replicate.from(COUCH_URL_POSTS)
    await commentsDB.value?.replicate.to(COUCH_URL_COMMENTS)
    await commentsDB.value?.replicate.from(COUCH_URL_COMMENTS)
    fetchData()
  } catch (e) {
    console.error('Erreur sync manuelle', e)
  }
}

// Fetch
const fetchData = async () => {
  if (!postsDB.value || !commentsDB.value) return
  try {
    const postsRes = await postsDB.value.allDocs({ include_docs: true })
    const commentsRes = await commentsDB.value.allDocs({ include_docs: true })

    const posts = postsRes.rows.map((r) => r.doc!).filter(Boolean)
    const comments = commentsRes.rows.map((r) => r.doc!).filter(Boolean)

    postsData.value = posts.map((post) => ({
      ...post,
      displayComments: comments.filter((c) => c.postId === post._id),
    }))
  } catch (err) {
    console.error('Erreur fetchData:', err)
  }
}

// CRUD Posts
const createDoc = (doc: Partial<Post>) => {
  const newDoc = { ...doc, likes: 0 }
  postsDB.value?.post(newDoc).then(fetchData)
}

const deleteDoc = (doc: Post) => {
  postsDB.value?.remove(doc._id!, doc._rev!).then(fetchData)
}

const updateDoc = (doc: Post) => {
  const { displayComments, ...clean } = doc
  postsDB.value?.put({ ...clean, ville: 'Update ' + new Date().toISOString() }).then(fetchData)
}

const toggleLike = (post: Post) => {
  const { displayComments, ...clean } = post
  postsDB.value?.put({ ...clean, likes: (clean.likes || 0) + 1 }).then(fetchData)
}

// Comments
const addComment = (post: Post) => {
  const txt = prompt('Commentaire :')
  if (!txt || !commentsDB.value) return

  const newComment: CommentDoc = {
    postId: post._id!,
    text: txt,
    date: new Date().toISOString(),
  }
  commentsDB.value.post(newComment).then(fetchData)
}

const deleteComment = (comment: CommentDoc) => {
  commentsDB.value?.remove(comment._id!, comment._rev!).then(fetchData)
}

const updateComment = (comment: CommentDoc) => {
  const newText = prompt('Modifier commentaire :', comment.text)
  if (!newText || !commentsDB.value) return
  commentsDB.value.put({ ...comment, text: newText }).then(fetchData)
}

// Recherche & Index
const createIndex = () => {
  postsDB.value?.createIndex({ index: { fields: ['likes'] } })
  postsDB.value?.createIndex({ index: { fields: ['nom'] } })
}

const searchByName = async () => {
  if (!searchTerm.value) return fetchData()
  if (!postsDB.value || !commentsDB.value) return

  try {
    const res = await postsDB.value.find({
      selector: { nom: { $regex: new RegExp(searchTerm.value, 'i') } },
    })
    const commentsRes = await commentsDB.value.allDocs({ include_docs: true })
    const comments = commentsRes.rows.map((r) => r.doc!).filter(Boolean)

    postsData.value = res.docs.map((post) => ({
      ...post,
      displayComments: comments.filter((c) => c.postId === post._id),
    }))
  } catch (err) {
    console.error('Erreur searchByName:', err)
  }
}

const sortByLikes = () => {
  postsData.value.sort((a, b) => (b.likes || 0) - (a.likes || 0))
}

// Factory
const createFactoryDocs = async () => {
  if (!postsDB.value || !commentsDB.value) return

  const sports = ['Football', 'Tennis', 'Rugby']
  const posts: Post[] = []
  const comments: CommentDoc[] = []

  for (let i = 0; i < 10; i++) {
    const id = `post_${Date.now()}_${i}`
    posts.push({
      _id: id,
      nom: `User ${i + 1}`,
      age: 20 + i,
      ville: `Ville ${i}`,
      sport: sports[i % 3],
      likes: Math.floor(Math.random() * 50),
    })
    comments.push({
      postId: id,
      text: `Commentaire ${i + 1}`,
      date: new Date().toISOString(),
    })
  }

  await postsDB.value.bulkDocs(posts)
  await commentsDB.value.bulkDocs(comments)
  fetchData()
}

// Mounted
onMounted(() => {
  initDatabases()
  createIndex()
  fetchData()
})
</script>

<template>
  <h1>Tp InfraDon - Posts et Commentaires</h1>

  <!-- Compteur -->
  <p>
    Compteur: {{ counter }}
    <button @click="increment">+1</button>
  </p>
  <hr />

  <!-- Connexion / Offline -->
  <div>
    <h2>Connexion</h2>
    <p>
      Etat: <strong>{{ isOffline ? 'HORS LIGNE' : 'EN LIGNE' }}</strong>
    </p>
    <button @click="toggleOfflineMode">Changer <=> Online/Offline</button>
    <button @click="MAJserveur">Sync Manuelle</button>
  </div>
  <hr />

  <!-- Outils -->
  <div>
    <h2>Outils</h2>
    <button @click="createFactoryDocs">Factory (2 collections)</button>
    <button @click="sortByLikes">Trier par Likes</button>
    <br />
    <input v-model="searchTerm" placeholder="Rechercher un nom..." />
    <button @click="searchByName">Rechercher</button>
    <button @click="fetchData">Reset la liste</button>
  </div>
  <hr />

  <!-- Liste des Posts -->
  <div>
    <h2>Liste des Posts</h2>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <h3>{{ post.nom }} ({{ post.likes }} likes)</h3>
        <p>{{ post.ville }} - {{ post.sport }}</p>

        <button @click="toggleLike(post)">Liker</button>
        <button @click="updateDoc(post)">Modifier le post</button>
        <button @click="deleteDoc(post)">Supprimer le post</button>
        <button @click="addComment(post)">Ajouter un commentaire</button>

        <!-- Liste des commentaires du post -->
        <div v-if="post.displayComments?.length">
          <h4>Commentaires :</h4>
          <ul>
            <li v-for="comment in post.displayComments" :key="comment._id">
              {{ comment.text }} — {{ comment.date }}
              <button @click="deleteComment(comment)">Supprimer le commentaire</button>
              <button @click="updateComment(comment)">Modifier le commentaire</button>
            </li>
          </ul>
        </div>
        <hr />
      </li>
    </ul>
  </div>

  <!-- Créer un post rapide -->
  <button @click="createDoc({ nom: 'Aissya', age: 20, ville: 'London', sport: 'Taichi' })">
    Créer Document
  </button>
</template>
