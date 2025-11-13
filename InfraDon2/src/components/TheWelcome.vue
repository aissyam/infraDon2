<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

const counter = ref(0)
const increment = () => {
  counter.value++
}

interface Post {
  nom: string
  age: number
  ville: string
  _id?: string
  _rev?: string
}

// Référence à la base de données
const storage = ref<PouchDB.Database | null>(null)
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const localdb = new PouchDB('ma_collection')
  //const remotedb = new PouchDB('http://admin:cestcatastrophiiique@localhost:5984/ma_collection')
  if (localdb) {
    console.log('Connecté à la collection : ' + localdb?.name)
    storage.value = localdb
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }

  // Réplicatiosn (unidirectionnelles)
  // localdb.replicate
  //   .from('http://admin:cestcatastrophiiique@localhost:5984/ma_collection', {
  //     live: true,
  //     retry: false,
  //   })
  //   .on('change', (info) => {
  //     console.log('Changements reçus du serveur', info)
  //     fetchData()
  //   })
  //   .on('complete', (info) => {
  //     console.log('Réplication initiale terminée', info)
  //   })
  //   .on('error', (err) => {
  //     console.error('Erreur réplication pull :', err)
  //   })

  // localdb.replicate
  //   .to('http://admin:cestcatastrophiiique@localhost:5984/ma_collection', {
  //     live: true,
  //     retry: false,
  //   })
  //   .on('change', (info) => {
  //     console.log('Changements reçus du serveur', info)
  //     fetchData()
  //   })
  //   .on('complete', (info) => {
  //     console.log('Réplication initiale terminée', info)
  //   })
  //   .on('error', (err) => {
  //     console.error('Erreur réplication pull :', err)
  //   })

  // Synchronisation
  localdb
    .sync('http://admin:cestcatastrophiiique@localhost:5984/ma_collection', {
      live: true,
      retry: false,
    })
    .on('change', (info) => {
      console.log('Synchronisation : changement détecté', info)
      fetchData()
    })
    .on('complete', (info) => {
      console.log('Synchronisation terminée', info)
    })
    .on('error', (err) => {
      console.log('Erreur de synchronisation :', err)
    })
}

// Récupération des données
const fetchData = () => {
  if (!storage.value) return
  storage.value
    .allDocs({
      include_docs: true,
      attachments: true,
    })
    .then((result) => {
      postsData.value = result.rows.map((row) => row.doc)
      console.log(postsData.value)
      // handle result
    })
    .catch(function (err) {
      console.log(err)
    })

  // TODO Récupération des données
  // https://pouchdb.com/api.html#batch_fetch
  // Regarder l'exemple avec function allDocs
  // Remplir le tableau postsData avec les données récupérées
}

const createDoc = (newDoc: Post) => {
  if (!storage.value) return
  storage.value
    .post(newDoc)
    .then((response) => {
      console.log(response)
      fetchData()
    })
    .catch(function (err) {
      console.log(err)
    })
}

const updateDoc = (doc: Post) => {
  const newDoc = { ...doc }
  newDoc.ville = 'Lausanne' // 'Update' + new Date().toISOString()
  if (!storage.value) return
  storage.value
    .put(newDoc)
    .then((response) => {
      console.log(response)
      fetchData()
    })
    .catch(function (err) {
      console.log(err)
    })
}

const deleteDoc = (doc: Post) => {
  if (!storage.value) return
  storage.value
    .remove(doc)
    .then((response) => {
      console.log(response)
      fetchData()
    })
    .catch(function (err) {
      console.log(err)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
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
    </li>
  </ul>
  <button @click="createDoc({ nom: 'Marie', age: 22, ville: 'Séoul' })">Créer</button>
</template>
