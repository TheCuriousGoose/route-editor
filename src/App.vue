<script setup>
import { ref, computed, defineExpose } from 'vue';
import RouteEditor from './components/RouteEditor.vue';
import ViewMode from './components/RouteViewer.vue';

const isViewMode = ref(false);
const editorRef = ref(null);

const elements = computed(() => editorRef.value ? editorRef.value.elements : []);
const links = computed(() => editorRef.value ? editorRef.value.links : []);

const toggleViewMode = () => {
  isViewMode.value = !isViewMode.value;
};

// Expose `editorRef` to allow access to `elements` and `links` in `ViewMode`
defineExpose({ editorRef });
</script>

<template>
  <button @click="toggleViewMode">{{ isViewMode ? 'Edit Mode' : 'View Mode' }}</button>
  <RouteEditor v-if="!isViewMode" ref="editorRef" />
  <ViewMode v-if="isViewMode" :elements="elements" :links="links" />
</template>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
