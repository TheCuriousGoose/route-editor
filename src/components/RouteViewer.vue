<script setup>
import { computed, ref } from 'vue';

const props = defineProps({
    elements: Array,
    links: Array,
});

const currentIndex = ref(0);

const currentElement = computed(() => {
    return props.elements[currentIndex.value];
});

const getLinksForElement = (index) => {
    return props.links.filter((link) => link.startElement === index);
};

const navigateTo = (index) => {
    currentIndex.value = index;
};
</script>

<template>
    <div class="view-mode">
        <div class="slide" v-if="currentElement">
            <h2>{{ currentElement.content }}</h2>
            <div class="navigation">
                <button v-for="(link, index) in getLinksForElement(currentIndex)" :key="index"
                    @click="navigateTo(link.endElement)">
                    {{ link.endElement + 1 }}
                </button>
            </div>
        </div>
    </div>
</template>

<style scoped lang="scss">
.view-mode {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    background: #333;
    color: white;
    text-align: center;

    .slide {
        width: 80%;
        height: 80%;
        background: #4caf50;
        border-radius: 10px;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        padding: 20px;

        .navigation {
            margin-top: 20px;

            button {
                margin: 0 10px;
                padding: 10px 20px;
                background: #e74c3c;
                color: white;
                border: none;
                border-radius: 5px;
                cursor: pointer;
            }
        }
    }
}
</style>
