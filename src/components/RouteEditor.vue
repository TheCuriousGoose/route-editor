<script setup>
import { ref, onMounted, watchEffect } from 'vue';

// State management
const elements = ref([
    {
        content: 'Root Element',
        position: { x: 300, y: 300 },
        showInteractions: false,
        activeQuadrants: new Map(), // To track active quadrants
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
    const parent = elements.value[parentIndex];

    // Define opposite quadrants
    const oppositeQuadrant = (dir) => {
        switch (dir) {
            case 1: return 4; // Top is opposite to Bottom
            case 2: return 3; // Left is opposite to Right
            case 3: return 2; // Right is opposite to Left
            case 4: return 1; // Bottom is opposite to Top
            default: return null;
        }
    };

    // Check if quadrant is already occupied
    if (parent.activeQuadrants.has(direction)) {
        alert('This quadrant already has a connection.');
        return;
    }
    const newElement = {
        content: 'New Element',
        position: calculateNewPosition(parentIndex, direction),
        showInteractions: false,
        activeQuadrants: new Map(),
    };
    elements.value.push(newElement);


    newElement.activeQuadrants.set(oppositeQuadrant(direction), elements.value.length)

    // Add the link
    links.value.push({
        start: elements.value[parentIndex].position,
        end: newElement.position,
        startElement: parentIndex,
        endElement: elements.value.length - 1,
    });

    // Mark the quadrant as active on the parent node
    parent.activeQuadrants.set(direction, elements.value.length - 1);
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
            x -= offset;
            break;
        case 4:
            y += offset;
            break;
    }
    return { x, y };
};

const deleteElement = (index) => {
    // Remove links connected to the deleted element
    links.value = links.value.filter(
        (link) => link.startElement !== index && link.endElement !== index
    );

    // Update parent elements to clear the quadrant
    elements.value.forEach((element) => {
        for (let [key, value] of element.activeQuadrants) {
            if (value === index) {
                element.activeQuadrants.delete(key);
            }
        }
    });

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
    console.log('test');
    if (editingElement.value !== null) {
        elements.value[editingElement.value].content = popupContent.value;
        editingElement.value = null;
    }
};

const cancelEdit = () => {
    editingElement.value = null;
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

        // Mark the quadrant as active
        startElement.activeQuadrants.set(linkingStart.value.interactionArea, endElementIndex);
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
        const targetIndex = event.target.dataset.index;
        deleteElement(targetIndex);
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
<template>
    <div class="route-editor" @mouseup="stopLinking">
        <!-- Render the links -->
        <svg class="link-canvas" xmlns="http://www.w3.org/2000/svg">
            <line v-for="(link, index) in links" :key="index" :x1="link.start.x" :y1="link.start.y" :x2="link.end.x"
                :y2="link.end.y" stroke="black" stroke-width="2" @click.stop="deleteLink(index)" class="link" />
        </svg>

        <!-- Render the elements -->
        <div class="element-wrapper" v-for="(element, index) in elements" :key="index"
            :style="{ top: element.position.y + 'px', left: element.position.x + 'px' }">

            <!-- Node -->
            <div class="element" @mousedown.stop="startDrag(index, $event)" tabindex="0"
                @mouseover="showInteractions(index)" @mouseleave="hideInteractions(index)">
                <div class="content">{{ element.content }}</div>
            </div>

            <!-- Revealed Quadrant Circle -->
            <div class="quadrant-circle" @mouseover="showInteractions(index)" @mouseleave="hideInteractions(index)"
                v-if="/*element.showInteractions*/ true">
                <div class="quadrant top" :class="{ active: element.activeQuadrants.has(1) }"
                    @click="createNode(index, 1)">
                    <span>1</span>
                </div>
                <div class="quadrant left" :class="{ active: element.activeQuadrants.has(2) }"
                    @click="createNode(index, 2)">
                    <span>2</span>
                </div>
                <div class="quadrant right" :class="{ active: element.activeQuadrants.has(3) }"
                    @click="createNode(index, 3)">
                    <span>3</span>
                </div>
                <div class="quadrant bottom" :class="{ active: element.activeQuadrants.has(4) }"
                    @click="createNode(index, 4)">
                    <span>4</span>
                </div>
            </div>

            <!-- Context Menu (Edit and Delete Buttons) -->
            <div v-if="element.showInteractions" class="context-menu" @mouseenter="element.showInteractions = true"
                @mouseleave="element.showInteractions = false">
                <button class="edit-btn" @click="editElement(index)">Edit</button>
                <button class="delete-btn" @click="deleteElement(index)">Delete</button>
            </div>
        </div>

        <!-- Popup for Editing -->
        <div v-if="editingElement !== null" class="popup">
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

    .element-wrapper {
        position: absolute;
        width: 100px;
        height: 100px;
        transform: translate(-50%, -50%);
    }

    .element {
        position: relative;
        width: 100px;
        height: 100px;
        border-radius: 50%;
        background-color: #4caf50;
        color: white;
        text-align: center;
        display: flex;
        justify-content: center;
        align-items: center;
        font-size: 14px;
        user-select: none;
        cursor: pointer;
        z-index: 1;
        transition: background-color 0.2s ease-in-out, opacity 0.2s ease-in-out;
    }

    .quadrant-circle {
        position: absolute;
        top: 50%;
        left: 50%;
        width: 100px;
        height: 100px;
        transform: translate(-50%, -50%);
        display: flex;
        justify-content: center;
        align-items: center;
        z-index: 2;
        pointer-events: none;

        .quadrant {
            position: absolute;
            width: 50px;
            height: 50px;
            background-color: #333;
            color: white;
            display: flex;
            justify-content: center;
            align-items: center;
            font-size: 16px;
            transition: background-color 0.2s ease-in-out, transform 0.1s ease;
            pointer-events: all; // Ensure quadrants are clickable
            z-index: 1;
            gap: 0;
            rotate: 45deg;
            border: 1px black solid;

            &:hover {
                background-color: #555;
            }

            &:active {
                background-color: #777;
                transform: scale(0.95);
            }

            &.active {
                background-color: #2ecc71; // Active color
            }

            &.top {
                border-radius: 100% 0 0 0;
                transform: translate(-35px, -35px);
            }

            &.left {
                border-radius: 0 100% 0 0;
                transform: translate(35px, -35px);
            }

            &.right {
                border-radius: 0 0 0 100%;
                transform: translate(-35px, 35px);
            }

            &.bottom {
                border-radius: 0 0 100% 0;
                transform: translate(35px, 35px);
            }

            span {
                rotate: -45deg;
            }
        }
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
            margin-bottom: 5px;
        }

        .edit-btn {
            background: #2ecc71;
        }

        .delete-btn {
            background: #e74c3c;
        }
    }

    .link-canvas {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        pointer-events: none;

        .link {
            pointer-events: all;
        }
    }

    .popup {
        position: fixed;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        background: white;
        border: 1px solid #ddd;
        box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
        z-index: 1000;
        width: 300px;
        padding: 20px;
        border-radius: 8px;

        .popup-content {
            display: flex;
            flex-direction: column;
            gap: 10px;

            h3 {
                margin: 0;
            }

            textarea {
                width: 100%;
                height: 100px;
                padding: 10px;
                border-radius: 4px;
                border: 1px solid #ccc;
            }

            button {
                padding: 10px;
                border: none;
                border-radius: 4px;
                cursor: pointer;

                &:first-child {
                    background-color: #2ecc71;
                    color: white;
                }

                &:last-child {
                    background-color: #e74c3c;
                    color: white;
                }
            }
        }
    }
}
</style>
