<script setup lang="ts">
import WelcomeItem from './WelcomeItem.vue'
import DocumentationIcon from './icons/IconDocumentation.vue'
import ToolingIcon from './icons/IconTooling.vue'
import EcosystemIcon from './icons/IconEcosystem.vue'
import CommunityIcon from './icons/IconCommunity.vue'
import SupportIcon from './icons/IconSupport.vue'
import { ref } from 'vue'
import PouchDB from 'pouchdb'
//const openReadmeInEditor = () => fetch('/__open-in-editor?file=README.md')

mounted() {
    this.initDatabase()
}



const counter = ref(110)

const increment ={} => {
  CountQueuingStrategy.value++
}
</script>

<template>
  <h1>Welcome</h1>
  <p>Counter {{ counter }}</p>
  <button @click="increment"></button>
</template>
