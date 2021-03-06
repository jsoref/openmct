<template>
<li
    ref="me"
    :style="{
        'top': virtualScroll ? itemTop : 'auto',
        'position': virtualScroll ? 'absolute' : 'relative'
    }"
    class="c-tree__item-h"
>
    <div
        class="c-tree__item"
        :class="{
            'is-alias': isAlias,
            'is-navigated-object': navigated
        }"
    >
        <view-control
            v-model="expanded"
            class="c-tree__item__view-control"
            :control-class="'c-nav__up'"
            :enabled="showUp"
            @input="resetTreeHere"
        />
        <object-label
            :domain-object="node.object"
            :object-path="node.objectPath"
            :navigate-to-path="navigateToPath"
            :style="{ paddingLeft: leftOffset }"
        />
        <view-control
            v-model="expanded"
            class="c-tree__item__view-control"
            :control-class="'c-nav__down'"
            :enabled="hasComposition && showDown"
        />
    </div>
</li>
</template>

<script>
import viewControl from '../components/viewControl.vue';
import ObjectLabel from '../components/ObjectLabel.vue';

export default {
    name: 'TreeItem',
    inject: ['openmct'],
    components: {
        viewControl,
        ObjectLabel
    },
    props: {
        node: {
            type: Object,
            required: true
        },
        leftOffset: {
            type: String,
            default: '0px'
        },
        showUp: {
            type: Boolean,
            default: false
        },
        showDown: {
            type: Boolean,
            default: true
        },
        itemIndex: {
            type: Number,
            required: false,
            default: undefined
        },
        itemOffset: {
            type: Number,
            required: false,
            default: undefined
        },
        itemHeight: {
            type: Number,
            required: false,
            default: 0
        },
        virtualScroll: {
            type: Boolean,
            default: false
        },
        shouldEmitHeight: {
            type: Boolean,
            default: false
        }
    },
    data() {
        this.navigateToPath = this.buildPathString(this.node.navigateToParent);

        return {
            hasComposition: false,
            navigated: this.navigateToPath === this.openmct.router.currentLocation.path,
            expanded: false
        };
    },
    computed: {
        isAlias() {
            let parent = this.node.objectPath[1];
            if (!parent) {
                return false;
            }

            let parentKeyString = this.openmct.objects.makeKeyString(parent.identifier);

            return parentKeyString !== this.node.object.location;
        },
        itemTop() {
            return (this.itemOffset + this.itemIndex) * this.itemHeight + 'px';
        }
    },
    watch: {
        expanded() {
            this.$emit('expanded', this.domainObject);
        }
    },
    mounted() {
        let objectComposition = this.openmct.composition.get(this.node.object);

        // only reliable way to get final item height
        this.readyStateCheck();

        this.domainObject = this.node.object;
        let removeListener = this.openmct.objects.observe(this.domainObject, '*', (newObject) => {
            this.domainObject = newObject;
        });

        this.$once('hook:destroyed', removeListener);
        if (objectComposition) {
            this.hasComposition = true;
        }

        this.openmct.router.on('change:path', this.highlightIfNavigated);
    },
    destroyed() {
        this.openmct.router.off('change:path', this.highlightIfNavigated);
        document.removeEventListener('readystatechange', this.emitHeight);
    },
    methods: {
        readyStateCheck() {
            if (document.readyState !== 'complete') {
                document.addEventListener('readystatechange', this.emitHeight);
            } else {
                this.emitHeight();
            }
        },
        emitHeight() {
            if (this.shouldEmitHeight && document.readyState === 'complete') {
                this.$emit('emittedHeight', this.$el.offsetHeight);
            }
        },
        buildPathString(parentPath) {
            return [parentPath, this.openmct.objects.makeKeyString(this.node.object.identifier)].join('/');
        },
        highlightIfNavigated(newPath, oldPath) {
            if (newPath === this.navigateToPath) {
                this.navigated = true;
            } else if (oldPath === this.navigateToPath) {
                this.navigated = false;
            }
        },
        resetTreeHere() {
            this.$emit('resetTree', this.node);
        }
    }
};
</script>
