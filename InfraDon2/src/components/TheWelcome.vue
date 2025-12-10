<script setup lang="ts">
import PouchDB from 'pouchdb'
import { onMounted, ref } from 'vue'
import PouchDBFind from "pouchdb-find";
PouchDB.plugin(PouchDBFind);

declare interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

// Connecter (ok)
// Synchro Distant vers local (ok)
// CRUD : create (post) read (alldocs) update delete -> TODO update delete
// Synchro Local vers Distant (ok)

// Ecouter les changements sur la base distante
/* function sync
storage.value.sync(distantDB).then(() => {
  console.log('Changement detecte')
  synchroDistantLocal()
})
*/

// Référence à la base de données
const storage = ref()

// Données stockées
const postsData = ref<Post[]>([])

const localDB = 'local_infra'
const distantDB = 'http://admin:admin@localhost:5984/database'

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')

  const db = new PouchDB(localDB)
  if (db) {
    console.log('Connecté à la collection : ' + db?.name)
    storage.value = db
    synchroDistantLocal()
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

const synchroDistantLocal = () => {
  storage.value.replicate.from(distantDB).then(() => {
    console.log('replication est termine')
    fetchData()
  })
}

const synchroLocalDistant = () => {
  storage.value.replicate.to(distantDB).then(() => {
    console.log('replication est termine')
    fetchData()
  })
}

// Create
const addDocument = () => {
  const doc = {
    post_name: 'rwe',
    post_content: 'rwe' + new Date().toISOString(),
  }
  storage.value.post(doc).then(() => {
    console.log('ajout ok')
    fetchData()
  })
}


// Récupération des données
const fetchData = () => {
  storage.value
    .allDocs({
      include_docs: true,
      attachments: true,
    })
    .then((result) => {
      console.log('result', result)
      postsData.value = result.rows.map((r) => r.doc)
      // handle result
    })
    .catch((err) => {
      console.log(err)
    })
}

// Update
const updateDocument = (id: string, updatedFields: any) => {
  storage.value.get(id).then((doc) => {
    Object.assign(doc, updatedFields)
    return storage.value.put(doc)
  }).then(fetchData)
}

//Delete
const deleteDocument = (id: string) => {
  storage.value.get(id).then((doc) => {
    return storage.value.remove(doc)
  }).then(fetchData)
}

//Indexation

const createIndex = () => {
  storage.value.createIndex({ index: { fields: ['post_name'] } })
}

//Fonction de recherche
const searchPosts = async (keyword) => {
  const result = await storage.value.find({
    selector: { post_name: { $regex: keyword } },
  })
  postsData.value = result.docs
}

//Mode Offline
const isOnline = ref(true)



// TODO Récupération des données
// https://pouchdb.com/api.html#batch_fetch
// Regarder l'exemple avec function allDocs
// Remplir le tableau postsData avec les données récupérées
// faire l'indexation pour la recherche (c'est quoi ? aucune idée)


onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  createIndex()
})

const counter = ref(110)

const increment = () => {
  counter.value++
}
</script>

<template>
  <div>
    <h1>Welcome</h1>
    <p>Counter: {{ counter }}</p>
    <button @click="increment">+1</button>
    <button @click="isOnline = !isOnline"> {{ isOnline ? 'Passer hors ligne' : 'Revenir en ligne' }}</button>
    <h1>Fetch Data</h1>
    <article v-for="post in postsData" :key="(post as any).id">
      <h2>{{ post.post_name }}</h2>
      <p>{{ post.post_content }}</p>
      <button @click="updateDocument(post._id, { post_name: 'nouveau titre' })">Modifier</button>
      <button @click="deleteDocument(post._id)">Supprimer</button>
    </article>
    <button @click="addDocument">add document</button>
    <button @click="synchroLocalDistant">Synchro</button>

  </div>
</template>
