<script setup lang="ts">
  import { onMounted, ref } from 'vue'
  import PouchDB from 'pouchdb'
  import PouchDBFind from "pouchdb-find";
  PouchDB.plugin(PouchDBFind);

  // INTERFACE POST
  declare interface Post {
    _id?: string
    _rev?: string
    post_name: string
    post_content: string
    likes?: number  //  likes
    comments?: Comment[]  //  commentaires
    attributes: {
      creation_date: any
    }
    _attachments?: any  // médias
  }

  // INTERFACE COMMENTAIRES
  declare interface Comment {
    _id?: string
    comment_text: string
    post_id: string
    creation_date: string
    author?: string
  }

  // Connecter (ok)
  // Synchro Distant vers local (ok)
  // CRUD : create (post) read (alldocs) update delete -> ok
  // Synchro Local vers Distant (ok)

  // Ecouter les changements sur la base distante -> ok maintenant

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
        postsData.value = result.rows.map((r) => r.doc).filter((doc) => !doc._id.startsWith("_"))
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
  const searchPosts = async (keyword: string) => {
    const result = await storage.value.find({
      selector: { post_name: { $regex: keyword } },
    })
    console.log(result.docs)
    console.log('result', result.docs)
    postsData.value = result.docs;
  }

  //Mode Offline
  const isOnline = ref(true)
  const searchKeyword = ref('')

  // NOUVELLES VARIABLES POUR LES FONCTIONNALITÉS AJOUTÉES
  const syncHandler = ref<any>(null)
  const commentsVisible = ref<{ [key: string]: boolean }>({})
  const topLikedPosts = ref<Post[]>([])

  // mode offline FONCTIONNEL
  const toggleOnlineMode = () => {
    isOnline.value = !isOnline.value

    if (isOnline.value) {
      console.log('Mode ONLINE activé - Synchronisation automatique')
      startSync()
    } else {
      console.log('Mode OFFLINE activé - Travail local uniquement')
      stopSync()
    }
  }

  // synchro automatique
  const startSync = () => {
    syncHandler.value = storage.value.sync(distantDB, {
      live: true,
      retry: true
    }).on('change', (info) => {
      console.log('Changement détecté:', info)
      fetchData()
    }).on('error', (err) => {
      console.error('Erreur de synchronisation:', err)
    })
  }

  const stopSync = () => {
    if (syncHandler.value) {
      syncHandler.value.cancel()
      syncHandler.value = null
    }
  }

  // systeme likes
  const likePost = async (postId: string) => {
    try {
      const doc = await storage.value.get(postId)
      doc.likes = (doc.likes || 0) + 1
      await storage.value.put(doc)

      if (isOnline.value) {
        await synchroLocalDistant()
      }
      fetchData()
    } catch (err) {
      console.error('Erreur lors du like:', err)
    }
  }

  const sortByLikes = () => {
    postsData.value = [...postsData.value].sort((a, b) => (b.likes || 0) - (a.likes || 0))
  }

  // système commentaires
  const addComment = async (postId: string, commentText: string) => {
    if (!commentText.trim()) return

    try {
      const doc = await storage.value.get(postId)

      if (!doc.comments) {
        doc.comments = []
      }

      const newComment: Comment = {
        _id: 'comment_' + Date.now(),
        comment_text: commentText,
        post_id: postId,
        creation_date: new Date().toISOString(),
        author: 'User'
      }

      doc.comments.push(newComment)
      await storage.value.put(doc)

      if (isOnline.value) {
        await synchroLocalDistant()
      }
      fetchData()
    } catch (err) {
      console.error('Erreur lors de l ajout du commentaire:', err)
    }
  }

  const deleteComment = async (postId: string, commentId: string) => {
    try {
      const doc = await storage.value.get(postId)
      doc.comments = doc.comments.filter((c: Comment) => c._id !== commentId)
      await storage.value.put(doc)

      if (isOnline.value) {
        await synchroLocalDistant()
      }
      fetchData()
    } catch (err) {
      console.error('Erreur lors de la suppression du commentaire:', err)
    }
  }

  const toggleComments = (postId: string) => {
    commentsVisible.value[postId] = !commentsVisible.value[postId]
  }

  // gestion medias
  const attachMedia = async (postId: string, file: File) => {
    if (!file) return

    try {
      const doc = await storage.value.get(postId)
      const blob = new Blob([await file.arrayBuffer()], { type: file.type })

      await storage.value.putAttachment(
        postId,
        file.name,
        doc._rev,
        blob,
        file.type
      )

      if (isOnline.value) {
        await synchroLocalDistant()
      }
      fetchData()
    } catch (err) {
      console.error('Erreur lors de l ajout du média:', err)
    }
  }

  const deleteMedia = async (postId: string, attachmentName: string) => {
    try {
      const doc = await storage.value.get(postId)
      await storage.value.removeAttachment(postId, attachmentName, doc._rev)

      if (isOnline.value) {
        await synchroLocalDistant()
      }
      fetchData()
    } catch (err) {
      console.error('Erreur lors de la suppression du média:', err)
    }
  }

  const getMediaUrl = (post: Post, attachmentName: string) => {
    if (post._attachments && post._attachments[attachmentName]) {
      const attachment = post._attachments[attachmentName]
      const blob = new Blob([attachment.data], { type: attachment.content_type })
      return URL.createObjectURL(blob)
    }
    return null
  }

  //factory generer test
  const generateFakeData = async (count: number = 10) => {
    console.log(`Génération de ${count} documents de test...`)

    const fakePosts = []
    for (let i = 0; i < count; i++) {
      fakePosts.push({
        post_name: `Post Test ${i + 1}`,
        post_content: `Contenu généré automatiquement le ${new Date().toISOString()}`,
        likes: Math.floor(Math.random() * 100),
        comments: [],
        attributes: {
          creation_date: new Date().toISOString()
        }
      })
    }

    try {
      await storage.value.bulkDocs(fakePosts)
      console.log('Documents de test créés avec succès')

      if (isOnline.value) {
        await synchroLocalDistant()
      }
      fetchData()
    } catch (err) {
      console.error('Erreur lors de la génération des données:', err)
    }
  }

  //  top 10 des post les plus liké
  const fetchTopLikedPosts = async (limit: number = 10) => {
    try {
      await storage.value.createIndex({
        index: { fields: ['likes'] }
      })

      const result = await storage.value.find({
        selector: { likes: { $gte: 0 } },
        sort: [{ 'likes': 'desc' }],
        limit: limit
      })

      topLikedPosts.value = result.docs
    } catch (err) {
      console.error('Erreur lors de la récupération des top posts:', err)
    }
  }

  const onInputChanged = (event) => {
    searchPosts(searchKeyword.value)
    console.log('event', event)
  }

  onMounted(() => {
    console.log('=> Composant initialisé')
    initDatabase()
    createIndex()
    startSync() // AJOUT: Démarrer la synchronisation automatique
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
      <input type="text" v-model="searchKeyword" @change="onInputChanged"></input>
      <button @click="increment">+1</button>

      <!-- bouton mode off -->
      <button @click="toggleOnlineMode"> {{ isOnline ? 'Passer hors ligne' : 'Revenir en ligne' }}</button>

      <!--  autres boutons -->
      <button @click="generateFakeData(20)">Générer 20 posts de test</button>
      <button @click="sortByLikes">Trier par likes</button>
      <button @click="fetchTopLikedPosts()">Top 10 posts likés</button>

      <h1>Fetch Data</h1>
      <ol>
        <li v-for="post in postsData" :key="(post as any)._id">
          <article>
            <h2>{{ post.post_name }}</h2>
            <p>{{ post.post_content }}</p>

            <!-- systeme de like -->
            <p>Likes: {{ post.likes || 0 }}</p>
            <button @click="likePost(post._id)">Liker</button>

            <button @click="updateDocument(post._id, { post_name: 'nouveau titre' })">Modifier</button>
            <button @click="deleteDocument(post._id)">Supprimer</button>

            <!-- systeme commentaire -->
            <div>
              <button @click="toggleComments(post._id)">
                {{ post.comments?.length || 0 }} commentaires
              </button>

              <div v-if="commentsVisible[post._id]">
                <input
                  :id="'comment-' + post._id"
                  type="text"
                  placeholder="Ajouter un commentaire..."
                />
                <button @click="addComment(post._id, (document.getElementById('comment-' + post._id) as HTMLInputElement).value); (document.getElementById('comment-' + post._id) as HTMLInputElement).value = ''">
                  Envoyer
                </button>

                <ul v-if="post.comments && post.comments.length > 0">
                  <li v-for="comment in post.comments" :key="comment._id">
                    {{ comment.author }}: {{ comment.comment_text }}
                    <button @click="deleteComment(post._id, comment._id)">Supprimer</button>
                  </li>
                </ul>
              </div>
            </div>

            <!-- gestion medias -->
            <div>
              <label>Ajouter un média:</label>
              <input
                :id="'file-' + post._id"
                type="file"
                @change="(e) => attachMedia(post._id, (e.target as HTMLInputElement).files[0])"
              />

              <div v-if="post._attachments">
                <div v-for="(attachment, name) in post._attachments" :key="name">
                  <img v-if="attachment.content_type.startsWith('image/')"
                       :src="getMediaUrl(post, name)"
                       style="max-width: 200px"
                  />
                  <button @click="deleteMedia(post._id, name)">Supprimer média</button>
                </div>
              </div>
            </div>
          </article>
        </li>
      </ol>
      <button @click="addDocument">add document</button>
      <button @click="synchroLocalDistant">Synchro</button>

      <!-- AFFICHAGE DES TOP POSTS  -->
      <div v-if="topLikedPosts.length > 0">
        <h2>Top 10 Posts les plus likés</h2>
        <ol>
          <li v-for="post in topLikedPosts" :key="post._id">
            {{ post.post_name }} - {{ post.likes || 0 }} likes
          </li>
        </ol>
      </div>
    </div>
  </template>