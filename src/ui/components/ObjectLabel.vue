<template>
<a
    class="c-tree__item__label c-object-label"
    :class="{
        classList,
        'is-missing': observedObject.status === 'missing'
    }"
    draggable="true"
    :href="objectLink"
    @dragstart="dragStart"
    @click="navigateOrPreview"
>
    <div
        class="c-tree__item__type-icon c-object-label__type-icon"
        :class="typeClass"
    >
        <span class="is-missing__indicator"
              title="This item is missing"
        ></span>
    </div>
    <div class="c-tree__item__name c-object-label__name">
        {{ observedObject.name }}
    </div>
</a>
</template>

<script>

import ObjectLink from '../mixins/object-link';
import ContextMenuGesture from '../mixins/context-menu-gesture';
import PreviewAction from '../preview/PreviewAction.js';

export default {
    mixins: [ObjectLink, ContextMenuGesture],
    inject: ['openmct'],
    props: {
        domainObject: {
            type: Object,
            required: true
        },
        objectPath: {
            type: Array,
            required: true
        },
        navigateToPath: {
            type: String,
            default: undefined
        }
    },
    data() {
        return {
            observedObject: this.domainObject
        };
    },
    computed: {
        classList() {
            const classList = this.observedObject.classList;
            if (!classList || !classList.length) {
                return '';
            }

            return classList.join(' ');
        },
        typeClass() {
            let type = this.openmct.types.get(this.observedObject.type);
            if (!type) {
                return 'icon-object-unknown';
            }

            return type.definition.cssClass;
        }
    },
    mounted() {
        if (this.observedObject) {
            let removeListener = this.openmct.objects.observe(this.observedObject, '*', (newObject) => {
                this.observedObject = newObject;
            });
            this.$once('hook:destroyed', removeListener);
        }

        this.previewAction = new PreviewAction(this.openmct);
    },
    methods: {
        navigateOrPreview(event) {
            if (this.openmct.editor.isEditing()) {
                event.preventDefault();
                this.preview();
            }
        },
        preview() {
            if (this.previewAction.appliesTo(this.objectPath)) {
                this.previewAction.invoke(this.objectPath);
            }
        },
        dragStart(event) {
            let navigatedObject = this.openmct.router.path[0];
            let serializedPath = JSON.stringify(this.objectPath);
            let keyString = this.openmct.objects.makeKeyString(this.domainObject.identifier);

            /*
             * Cannot inspect data transfer objects on dragover/dragenter so impossible to determine composability at
             * that point. If dragged object can be composed by navigated object, then indicate with presence of
             * 'composable-domain-object' in data transfer
             */
            if (this.openmct.composition.checkPolicy(navigatedObject, this.observedObject)) {
                event.dataTransfer.setData("openmct/composable-domain-object", JSON.stringify(this.domainObject));
            }

            // serialize domain object anyway, because some views can drag-and-drop objects without composition
            // (eg. notabook.)
            event.dataTransfer.setData("openmct/domain-object-path", serializedPath);
            event.dataTransfer.setData(`openmct/domain-object/${keyString}`, this.domainObject);
        }
    }
};
</script>
