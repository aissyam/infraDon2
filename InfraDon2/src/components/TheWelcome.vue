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
  postId: string
  text: string
  date: string
  _id?: string
  _rev?: string
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

const postsDB = ref<PouchDB.Database | undefined>()
const commentsDB = ref<PouchDB.Database | undefined>()
const postsData = ref<Post[]>([])

const searchTerm = ref('')
let postSyncHandler: PouchDB.Replication.Sync<any> | null = null
let commentSyncHandler: PouchDB.Replication.Sync<any> | null = null
const COUCH_URL_POSTS = 'http://admin:cestcatastrophiiique@localhost:5984/ma_collection'
const COUCH_URL_COMMENTS = 'http://admin:cestcatastrophiiique@localhost:5984/second'
//du cpoup la cest la nouvelle collections

// Mode offline
const isOffline = ref(false)

// Init
const initDatabases = () => {
  console.log('=> Init des 2 Collections')
  postsDB.value = new PouchDB('Posts')
  commentsDB.value = new PouchDB('Comments')
  if (!isOffline.value) {
    startLiveReplication()
  }
}

const startLiveReplication = () => {
  if (postSyncHandler || !postsDB.value || !commentsDB.value) return
  postSyncHandler = postsDB.value
    .sync(COUCH_URL_POSTS, { live: true, retry: true })
    .on('change', () => fetchData())
  commentSyncHandler = commentsDB.value
    .sync(COUCH_URL_COMMENTS, { live: true, retry: true })
    .on('change', () => fetchData())
}

const toggleOfflineMode = () => {
  isOffline.value = !isOffline.value
  if (isOffline.value) {
    postSyncHandler?.cancel()
    postSyncHandler = null
    commentSyncHandler?.cancel()
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
    await commentsDB.value?.replicate
      .to(COUCH_URL_COMMENTS)
      .catch((e) => console.log('Pas de DB remote comments'))
    await commentsDB.value?.replicate
      .from(COUCH_URL_COMMENTS)
      .catch((e) => console.log('Pas de DB remote comments'))
    fetchData()
  } catch (err) {
    console.error('Erreur sync manuelle:', err)
  }
}

// Fetch
const fetchData = async () => {
  if (!postsDB.value || !commentsDB.value) return
  try {
    const allPosts = await postsDB.value.allDocs({ include_docs: true })
    const posts = allPosts.rows.map((row: any) => row.doc)
    const allComments = await commentsDB.value.allDocs({ include_docs: true })
    const comments = allComments.rows.map((row: any) => row.doc)
    postsData.value = posts.map((post: Post) => {
      const linkedComments = comments.filter((c: CommentDoc) => c.postId === post._id)
      return { ...post, displayComments: linkedComments }
    })
    console.log('Données chargées et jointes.')
  } catch (err) {
    console.error('Erreur fetchData:', err)
  }
}

// CRUD
const createDoc = (newDoc: any) => {
  newDoc.likes = 0
  postsDB.value?.post(newDoc).then(() => fetchData())
}

const deleteDoc = (doc: Post) => {
  // Suppression du post
  postsDB.value?.remove(doc._id!, doc._rev!).then(() => fetchData())
}

const updateDoc = (doc: Post) => {
  const { displayComments, ...docToSave } = doc
  const docUpdated = { ...docToSave, ville: 'Update ' + new Date().toISOString() }
  postsDB.value?.put(docUpdated).then(() => fetchData())
}

const toggleLike = (post: Post) => {
  const { displayComments, ...docToSave } = post
  const updated = { ...docToSave, likes: (docToSave.likes || 0) + 1 }
  postsDB.value?.put(updated).then(() => fetchData())
}

const addComment = (post: Post) => {
  const txt = prompt('Commentaire :')
  if (!txt || !commentsDB.value) return
  const newComment: CommentDoc = {
    postId: post._id!,
    text: txt,
    date: new Date().toISOString(),
  }
  commentsDB.value
    .post(newComment)
    .then(() => {
      console.log('Commentaire ajouté dans la collection Comments')
      fetchData()
    })
    .catch((err) => console.error(err))
}
const deleteComment = (comment: CommentDoc) => {
  if (!commentsDB.value || !comment._id || !comment._rev) return
  commentsDB.value.remove(comment._id, comment._rev).then(() => fetchData())
}

// Index
const createIndex = () => {
  postsDB.value?.createIndex({ index: { fields: ['likes'] } })
  postsDB.value?.createIndex({ index: { fields: ['nom'] } })
}

const sortByLikes = () => {
  postsDB.value
    ?.find({
      selector: { likes: { $gte: 0 } },
      sort: [{ likes: 'desc' }],
    })
    .then((res: any) => {
      const sortedPosts = res.docs
      //pour recuperer pour lier le commentaires ua posts
      commentsDB.value?.allDocs({ include_docs: true }).then((comRes: any) => {
        const allComments = comRes.rows.map((r: any) => r.doc)
        postsData.value = sortedPosts.map((post: Post) => {
          const linked = allComments.filter((c: CommentDoc) => c.postId === post._id)
          return { ...post, displayComments: linked }
        })
      })
    })
}

const searchByName = () => {
  if (!searchTerm.value) {
    fetchData()
    return
  }
  const regex = new RegExp(`^${searchTerm.value}`, 'i')
  postsDB.value
    ?.find({
      selector: { nom: { $regex: regex } },
    })
    .then((res: any) => {
      const foundPosts = res.docs
      commentsDB.value?.allDocs({ include_docs: true }).then((comRes: any) => {
        const allComments = comRes.rows.map((r: any) => r.doc)
        postsData.value = foundPosts.map((post: Post) => {
          const linked = allComments.filter((c: CommentDoc) => c.postId === post._id)
          return { ...post, displayComments: linked }
        })
      })
    })
}

const createFactoryDocs = async () => {
  if (!postsDB.value || !commentsDB.value) return
  const postsToAdd: any[] = []
  const commentsToAdd: any[] = []
  const sports = ['Football', 'Tennis', 'Rugby']
  for (let i = 0; i < 10; i++) {
    const postId = 'post_factory_' + Date.now() + '_' + i
    const docPost = {
      _id: postId,
      nom: `User ${i + 1}`,
      age: 20 + i,
      ville: `Ville ${i}`,
      sport: sports[i % 3],
      likes: Math.floor(Math.random() * 50),
    }
    postsToAdd.push(docPost)

    const docCom = {
      postId: postId,
      text: `Super commentaire pour le post ${i}`,
      date: new Date().toISOString(),
    }
    commentsToAdd.push(docCom)
  }
  //mettre du coup dans les 2 collections nromalement
  await postsDB.value.bulkDocs(postsToAdd)
  await commentsDB.value.bulkDocs(commentsToAdd)
  console.log('Factory : Posts et Commentaires créés dans 2 collections.')
  fetchData()
}

onMounted(() => {
  initDatabases()
  createIndex()
  fetchData()
})
</script>
<template>
  <h1>Tp InfraDon - Posts et Comms</h1>
  <p>Compteur: {{ counter }} <button @click="increment">+1</button></p>
  <hr />
<<<<<<< HEAD

  <h2>Collection Cities</h2>

  <input v-model="searchCity" @input="searchCitiesDB" placeholder="Rechercher une ville..." />

  <ul>
    <li v-for="c in citiesData" :key="c._id">
      {{ c.nom }} — {{ c.pays }} — {{ c.population }} habitants
      <button @click="updateCity(c)">Modifier</button>
      <button @click="deleteCity(c)">Supprimer</button>
    </li>
  </ul>

  <button @click="createCity({ nom: 'Séoul', pays: 'South Korea', population: 9500000 })">
    Ajouter ville
=======
  <div>
    <h2>Connexion</h2>
    <p>
      Etat: <strong>{{ isOffline ? 'HORS LIGNE' : 'EN LIGNE' }}</strong>
    </p>
    <button @click="toggleOfflineMode">Changer <=> Online Offline</button>
    <button @click="MAJserveur">Sync Manuelle</button>
  </div>
  <hr />
  <div>
    <h2>Outils</h2>
    <button @click="createFactoryDocs">Factory (Avec implémentation 2 collections)</button>
    <button @click="sortByLikes">Trier par Likes (DB)</button>
    <br />
    <input v-model="searchTerm" placeholder="Nom..." />
    <button @click="searchByName">Rechercher</button>
    <button @click="fetchData">Reset Liste</button>
  </div>
  <hr />
  <div>
    <h2>Liste des Posts</h2>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <h3>{{ post.nom }} ({{ post.likes }} likes)</h3>
        <p>{{ post.ville }} - {{ post.sport }}</p>
        <button @click="toggleLike(post)">Liker</button>
        <button @click="updateDoc(post)">Modifier Post</button>
        <button @click="deleteDoc(post)">Supprimer Post</button>
        <button @click="addComment(post)">Ajouter Commentaire</button>
        <div v-if="post.displayComments && post.displayComments.length > 0">
          <h4>Commentaires:</h4>
          <ul>
            <li v-for="comment in post.displayComments" :key="comment._id">
              {{ comment.text }}
              <button @click="deleteComment(comment)">Suppr</button>
            </li>
          </ul>
        </div>
        <hr />
      </li>
    </ul>
  </div>
  <button @click="createDoc({ nom: 'Aissya', age: 20, ville: 'Séoul', sport: 'taichi' })">
    Créer Document
>>>>>>> da91252 (Résolution du problème merge)
  </button>
</template>
