<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

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

let postsChanges: any = null
let postsSync: any = null

interface City {
  nom: string
  pays: string
  population: number
  _id?: string
  _rev?: string
}

const citiesDB = ref<PouchDB.Database<City> | null>(null)
const citiesData = ref<City[]>([])

let citiesChanges: any = null
let citiesSync: any = null

const isOffline = ref(false)

const initDatabases = () => {
  storage.value = new PouchDB<Post>('ma_collection')
  citiesDB.value = new PouchDB<City>('second')

  console.log('Connecté à la 1ère DB :', storage.value.name)
  console.log('Connecté à la 2e DB :', citiesDB.value.name)
}

// Index
const createIndexes = async () => {
  await storage.value?.createIndex({ index: { fields: ['nom'] } })
  await citiesDB.value?.createIndex({ index: { fields: ['nom'] } })
}

const fetchPosts = () => {
  storage.value
    ?.allDocs({ include_docs: true })
    .then((result) => (postsData.value = result.rows.map((r) => r.doc!)))
}

const fetchCities = () => {
  citiesDB.value
    ?.allDocs({ include_docs: true })
    .then((result) => (citiesData.value = result.rows.map((r) => r.doc!)))
}

// Watcher
const startWatchers = () => {
  // POSTS
  postsChanges?.cancel()
  postsChanges = storage.value
    ?.changes({
      live: true,
      include_docs: true,
      since: 'now',
    })
    .on('change', fetchPosts)

  // CITIES
  citiesChanges?.cancel()
  citiesChanges = citiesDB.value
    ?.changes({
      live: true,
      include_docs: true,
      since: 'now',
    })
    .on('change', fetchCities)
}

// Synchro

const remotePosts = 'http://admin:cestcatastrophiiique@localhost:5984/ma_collection'
const remoteCities = 'http://admin:cestcatastrophiiique@localhost:5984/second'

const startSync = () => {
  postsSync?.cancel()
  citiesSync?.cancel()

  postsSync = storage.value?.sync(remotePosts, { live: true, retry: true })
  citiesSync = citiesDB.value?.sync(remoteCities, { live: true, retry: true })
}

const stopSync = () => {
  postsSync?.cancel()
  citiesSync?.cancel()

  postsSync = null
  citiesSync = null

  console.log('Synchronisation OFFLINE')
}

// --- TOGGLE ---
const toggleOffline = () => {
  isOffline.value = !isOffline.value

  if (isOffline.value) {
    stopSync()
  } else {
    startSync()
  }
}

// CRUD
const createPost = (doc: Post) => storage.value?.post(doc).then(fetchPosts)
const deletePost = (doc: Post) => storage.value?.remove(doc).then(fetchPosts)
const updatePost = (doc: Post) => {
  const updatedPost = { ...doc, ville: 'Lausanne' }
  storage.value?.put(updatedPost).then(fetchPosts)
}

const createCity = (city: City) => citiesDB.value?.post(city).then(fetchCities)
const deleteCity = (city: City) => citiesDB.value?.remove(city).then(fetchCities)
const updateCity = (city: City) => {
  const updatedCity = { ...city, population: 10000000 }
  citiesDB.value?.put(updatedCity).then(fetchPosts)
}

// Recherche avec l'Index
const searchPost = ref('')
const searchCity = ref('')

const searchPostsDB = async () => {
  if (!searchPost.value.trim()) return fetchPosts()
  const res = await storage.value?.find({
    selector: { nom: { $regex: RegExp(searchPost.value, 'i') } },
  })
  postsData.value = res?.docs ?? []
}

const searchCitiesDB = async () => {
  if (!searchCity.value.trim()) return fetchCities()
  const res = await citiesDB.value?.find({
    selector: { nom: { $regex: RegExp(searchCity.value, 'i') } },
  })
  citiesData.value = res?.docs ?? []
}

onMounted(() => {
  initDatabases()
  createIndexes()
  fetchPosts()
  fetchCities()
  startWatchers()
  startSync()
})
</script>

<template>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">+1</button>
  <button @click="toggleOffline">
    {{ isOffline ? 'Passer online' : 'Passer offline' }}
  </button>
  <h2>Collection Posts</h2>

  <input v-model="searchPost" @input="searchPostsDB" placeholder="Rechercher un post..." />

  <ul>
    <li v-for="p in postsData" :key="p._id">
      {{ p.nom }} - {{ p.ville }}
      <button @click="updatePost(p)">Modifier</button>
      <button @click="deletePost(p)">Supprimer</button>
    </li>
  </ul>

  <button @click="createPost({ nom: 'Aissya', age: 20, ville: 'Nairobi' })">Créer Post</button>

  <hr />

  <h2>Collection Cities</h2>

  <input v-model="searchCity" @input="searchCitiesDB" placeholder="Rechercher une ville..." />

  <ul>
    <li v-for="c in citiesData" :key="c._id">
      {{ c.nom }} — {{ c.pays }} — {{ c.population }} habitants
      <button @click="updateCity(c)">Modifier</button>
      <button @click="deleteCity(c)">Supprimer</button>
    </li>
  </ul>

  <button @click="createCity({ nom: 'London', pays: 'UK', population: 9000000 })">
    Ajouter ville
  </button>
</template>
