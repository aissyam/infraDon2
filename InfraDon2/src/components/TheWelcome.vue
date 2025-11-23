<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

const counter = ref(0)
const increment = () => counter.value++

interface Post {
  nom: string
  age: number
  ville: string
  _id?: string
  _rev?: string
}

// Base de données
const storage = ref<PouchDB.Database<Post> | null>(null)
const postsData = ref<Post[]>([])

const isOffline = ref(false)
let syncHandler: PouchDB.Replication.Sync<Post> | null = null
let changesHandler: PouchDB.Core.Changes<Post> | null = null

const initDatabase = () => {
  const localdb = new PouchDB<Post>('ma_collection')
  storage.value = localdb

  console.log('Connecté à la DB :', localdb.name)
}

const fetchData = () => {
  if (!storage.value) return

  storage.value
    .allDocs({ include_docs: true })
    .then((result) => {
      postsData.value = result.rows.map((r) => r.doc).filter(Boolean) as Post[]
    })
    .catch((err) => console.error(err))
}

// Watcher
const startChangesWatcher = () => {
  if (!storage.value) return

  if (changesHandler) {
    changesHandler.cancel()
    changesHandler = null
  }

  changesHandler = storage.value
    .changes({
      live: true,
      include_docs: true,
      since: 'now',
      skipSetup: true,
    })
    .on('change', () => fetchData())
    .on('error', (err) => console.error('Erreur Changes:', err))
}

// Synchro online
const startSync = () => {
  if (!storage.value) return

  const remoteURL = 'http://admin:cestcatastrophiiique@localhost:5984/ma_collection'

  if (syncHandler) {
    syncHandler.cancel()
    syncHandler = null
  }

  syncHandler = storage.value
    .sync(remoteURL, { live: true, retry: true })
    .on('change', (info) => {
      console.log('Synchronisation : changement détecté', info)
      fetchData()
    })
    .on('error', (err) => console.error('Erreur de synchronisation :', err))
}

// Synchro offline
const stopSync = () => {
  if (syncHandler) {
    syncHandler.cancel()
    syncHandler = null
    console.log('Mode offline : Synchro arrêtée')
  }
}

// Toggle
const toggleOffline = () => {
  isOffline.value = !isOffline.value

  if (isOffline.value) {
    stopSync()
  } else {
    startSync()
  }
}

// CRUD
const createDoc = (newDoc: Post) => {
  if (!storage.value) return

  storage.value
    .post(newDoc)
    .then(() => fetchData())
    .catch((err) => console.error(err))
}

const updateDoc = (doc: Post) => {
  if (!storage.value) return

  const updated = { ...doc, ville: 'Lausanne' }

  storage.value
    .put(updated)
    .then(() => fetchData())
    .catch((err) => console.error(err))
}

const deleteDoc = (doc: Post) => {
  if (!storage.value) return

  storage.value
    .remove(doc)
    .then(() => fetchData())
    .catch((err) => console.error(err))
}

onMounted(() => {
  console.log('=> Composant initialisé')

  initDatabase()
  fetchData()
  startChangesWatcher()
  startSync()
})
</script>

<template>
  <h1>Anyeonghaseyo ^^</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">+1</button>
  <p>PostDatas</p>
  <ul>
    <li v-for="(post, index) in postsData" :key="post._id ?? index">
      {{ post.nom }} - {{ post.age }} - {{ post.ville }}
      <button @click="updateDoc(post)">Modifier</button>
      <button @click="deleteDoc(post)">Supprimer</button>
      <button @click="toggleOffline">{{ isOffline ? 'Passer Online' : 'Passer Offline' }}</button>
    </li>
  </ul>
  <button @click="createDoc({ nom: 'Marie', age: 22, ville: 'Tokyo' })">Créer</button>
</template>
