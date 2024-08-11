<script setup>
import { ref, reactive, onMounted, watchEffect } from 'vue';

// State management
const elements = ref([
    {
        content: 'Root Element',
        position: { x: 300, y: 300 },
        showInteractions: false,
    },
]);

const links = ref([]);
const draggingElement = ref(null);
const linking = ref(false);
const linkingStart = ref(null);
const editingElement = ref(null);
const popupContent = ref('');

// Methods
const showInteractions = (index) => {
    elements.value[index].showInteractions = true;
};

const hideInteractions = (index) => {
    elements.value[index].showInteractions = false;
};

const createNode = (parentIndex, direction) => {
    const newElement = {
        content: 'New Element',
        position: calculateNewPosition(parentIndex, direction),
        showInteractions: false,
    };
    elements.value.push(newElement);

    links.value.push({
        start: elements.value[parentIndex].position,
        end: newElement.position,
        startElement: parentIndex,
        endElement: elements.value.length - 1,
    });
};

const calculateNewPosition = (parentIndex, direction) => {
    const parent = elements.value[parentIndex];
    const offset = 150;
    let x = parent.position.x;
    let y = parent.position.y;
    switch (direction) {
        case 1:
            y -= offset;
            break;
        case 2:
            x += offset;
            break;
        case 3:
            y += offset;
            break;
        case 4:
            x -= offset;
            break;
    }
    return { x, y };
};

const deleteElement = (index) => {
    links.value = links.value.filter(
        (link) => link.startElement !== index && link.endElement !== index
    );
    elements.value.splice(index, 1);
};

const startDrag = (index, event) => {
    draggingElement.value = index;
    document.addEventListener('mousemove', drag);
    document.addEventListener('mouseup', stopDrag);
};

const drag = (event) => {
    if (draggingElement.value !== null) {
        const element = elements.value[draggingElement.value];
        element.position.x = event.clientX - 50; // Adjust to center the element
        element.position.y = event.clientY - 50; // Adjust to center the element
        updateLinks(draggingElement.value);
    }
};

const stopDrag = () => {
    draggingElement.value = null;
    document.removeEventListener('mousemove', drag);
    document.removeEventListener('mouseup', stopDrag);
};

const editElement = (index) => {
    editingElement.value = index;
    popupContent.value = elements.value[index].content;
};

const saveEdit = () => {
    if (editingElement.value !== null) {
        elements.value[editingElement.value].content = popupContent.value;
        editingElement.value = null;
    }
};

const cancelEdit = () => {
    editingElement.value = null;
};

const startLinking = (index, interactionArea, event) => {
    if (event.shiftKey) {
        linking.value = true;
        linkingStart.value = {
            elementIndex: index,
            interactionArea: interactionArea,
            x: event.clientX,
            y: event.clientY,
        };
        document.addEventListener('mousemove', updateLinking);
        document.addEventListener('mouseup', stopLinking);
    }
};

const updateLinking = (event) => {
    if (!linking.value) return;

    const startElement = elements.value[linkingStart.value.elementIndex];
    const endElementIndex = findElementAtPosition(event.clientX, event.clientY);

    if (endElementIndex !== -1 &&
        endElementIndex !== linkingStart.value.elementIndex) {
        const endElement = elements.value[endElementIndex];

        links.value.push({
            start: {
                x: startElement.position.x,
                y: startElement.position.y,
            },
            end: endElement.position,
            startElement: linkingStart.value.elementIndex,
            endElement: endElementIndex,
        });
    }
};

const stopLinking = () => {
    linking.value = false;
    linkingStart.value = null;
    document.removeEventListener('mousemove', updateLinking);
    document.removeEventListener('mouseup', stopLinking);
};

const findElementAtPosition = (x, y) => {
    return elements.value.findIndex(
        (element) =>
            x >= element.position.x &&
            x <= element.position.x + 100 &&
            y >= element.position.y &&
            y <= element.position.y + 100
    );
};

const deleteLink = (index) => {
    links.value.splice(index, 1);
};

const updateLinks = (elementIndex) => {
    links.value.forEach((link) => {
        if (link.startElement === elementIndex) {
            link.start.x = elements.value[elementIndex].position.x;
            link.start.y = elements.value[elementIndex].position.y;
        }
        if (link.endElement === elementIndex) {
            link.end.x = elements.value[elementIndex].position.x;
            link.end.y = elements.value[elementIndex].position.y;
        }
    });
};

const handleKeydown = (event) => {
    if (event.key === 'Delete' || event.key === 'Backspace') {
        links.value = links.value.filter((link) => {
            return !(
                link.startElement === event.target.dataset.index ||
                link.endElement === event.target.dataset.index
            );
        });
    }
};

watchEffect(() => {
    if (draggingElement.value !== null) {
        updateLinks(draggingElement.value);
    }
});

onMounted(() => {
    document.addEventListener('keydown', handleKeydown);
});
</script>
\<template>
    <div class="route-editor" @mouseup="stopLinking">
        <!-- Render the links -->
        <svg class="link-canvas" xmlns="http://www.w3.org/2000/svg">
            <line v-for="(link, index) in links" :key="index" :x1="link.start.x" :y1="link.start.y" :x2="link.end.x"
                :y2="link.end.y" stroke="black" stroke-width="2" @click.stop="deleteLink(index)" class="link" />
        </svg>

        <!-- Render the elements -->
        <div class="element" v-for="(element, index) in elements" :key="index"
            :style="{ top: element.position.y + 'px', left: element.position.x + 'px' }"
            @mouseenter="showInteractions(index)" @mouseleave="hideInteractions(index)"
            @mousedown.stop="startDrag(index, $event)" tabindex="0">
            <div class="content">{{ element.content }}</div>

            <!-- Context Menu (Edit and Delete Buttons) -->
            <div v-if="element.showInteractions" class="context-menu">
                <button class="edit-btn" @click="editElement(index)">Edit</button>
                <button class="delete-btn" @click="deleteElement(index)">Delete</button>
            </div>

            <!-- Interaction Areas -->
            <div v-if="element.showInteractions" class="interaction-areas">
                <div class="interaction-area top-left" @click="createNode(index, 1)">1</div>
                <div class="interaction-area top-right" @click="createNode(index, 2)">2</div>
                <div class="interaction-area bottom-left" @click="createNode(index, 3)">3</div>
                <div class="interaction-area bottom-right" @click="createNode(index, 4)">4</div>
            </div>
        </div>

        <!-- Popup for editing -->
        <div v-if="editingElement !== null" class="popup" @click.stop>
            <div class="popup-content">
                <h3>Edit Element</h3>
                <textarea v-model="popupContent"></textarea>
                <button @click="saveEdit">Save</button>
                <button @click="cancelEdit">Cancel</button>
            </div>
        </div>
    </div>
</template>

<style scoped lang="scss">
.route-editor {
    position: relative;
    width: 100%;
    height: 100vh;
    background: #f9f9f9;
    overflow: hidden;

    .element {
        position: absolute;
        width: 100px;
        height: 100px;
        border-radius: 50%;
        /* Circular elements */
        padding: 20px;
        background-color: #4caf50;
        color: white;
        text-align: center;
        cursor: grab;
        display: flex;
        justify-content: center;
        align-items: center;
        font-size: 14px;
        user-select: none;
        box-sizing: border-box;
        transform: translate(-50%, -50%);
        /* Center the element around the cursor */

        &:hover .interaction-areas {
            display: flex;
        }

        &:hover .context-menu {
            display: flex;
        }

        .context-menu {
            position: absolute;
            top: 0;
            left: 100%;
            display: flex;
            flex-direction: column;
            background: #fff;
            border: 1px solid #ddd;
            border-radius: 4px;
            padding: 10px;
            box-shadow: 0 0 5px rgba(0, 0, 0, 0.2);
            z-index: 10;

            .edit-btn,
            .delete-btn {
                background: #333;
                color: white;
                border: none;
                padding: 5px 10px;
                border-radius: 5px;
                cursor: pointer;
            }

            .edit-btn {
                background: #2ecc71;
            }

            .delete-btn {
                background: #e74c3c;
            }
        }

        .interaction-areas {
            position: absolute;
            display: none;
            /* Hide by default */
            width: 100px;
            height: 100px;
            border-radius: 50%;
            pointer-events: none;
            z-index: 1;
            display: flex;
            justify-content: space-between;
            align-items: space-between;

            .interaction-area {
                position: absolute;
                width: 30px;
                height: 30px;
                background: #333;
                color: white;
                display: flex;
                justify-content: center;
                align-items: center;
                border-radius: 50%;
                cursor: pointer;
                z-index: 40;

                &.top-left {
                    top: 10%;
                    left: 10%;
                }

                &.top-right {
                    top: 10%;
                    right: 10%;
                }

                &.bottom-left {
                    bottom: 10%;
                    left: 10%;
                }

                &.bottom-right {
                    bottom: 10%;
                    right: 10%;
                }
            }
        }
    }

    .link-canvas {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        pointer-events: none;
        z-index: -1;

        .link {
            cursor: pointer;
        }
    }

    .popup {
        position: fixed;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        width: 400px;
        padding: 20px;
        background: white;
        box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
        border-radius: 8px;
        z-index: 1000;

        .popup-content {
            display: flex;
            flex-direction: column;
            align-items: center;

            h3 {
                margin-bottom: 10px;
            }

            textarea {
                width: 100%;
                height: 100px;
                margin-bottom: 10px;
            }

            button {
                margin: 5px;
                padding: 10px 20px;
                border: none;
                border-radius: 5px;
                cursor: pointer;
            }
        }
    }
}
</style>
