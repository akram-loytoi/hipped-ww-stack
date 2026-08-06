<template>
    <draggable
        v-model="internalItems"
        :item-key="itemKey"
        :clone="(el) => el"
        :group="group"
        :sort="sortable"
        :handle="handle?.length ? `.${handle}` : null"
        :disabled="isEditing || isReadonly"
        class="ww-stack-draggable"
        :class="{ 'ww-stack-draggable--collapsed': collapsed }"
        @change="onChange"
        @start="setDrag(true)"
        @end="setDrag(false)"
    >
        <template #header>
            <div v-if="!hideHeader" class="ww-stack-header">
                <wwLayout v-if="collapsed" path="collapsedHeaderElement"></wwLayout>
                <wwLayout v-else path="headerElement"></wwLayout>
            </div>
        </template>
        <template #item="{ element, index: itemIndex }">
            <div class="draggable-item">
                <wwLayoutItemContext
                    :index="itemIndex"
                    :item="null"
                    is-repeat
                    :data="element"
                    :repeated-items="internalItems"
                >
                    <wwLayout path="itemElement"></wwLayout>
                </wwLayoutItemContext>
            </div>
        </template>
        <template #footer>
            <wwLayout v-if="!hideFooter" path="footerElement" class="ww-stack-footer"></wwLayout>
        </template>
    </draggable>
</template>

<script>
import draggable from "vuedraggable";
import { watch } from "vue";

export default {
    components: {
        draggable,
    },
    inject: {
        customHandler: { defaultValue: null },
        customDragHandler: { defaultValue: null },
        customCollapseHandler: { defaultValue: null },
    },
    props: {
        wwElementState: { type: Object, required: true },
        content: { type: Object, required: true },
        uid: { type: String, required: true },
        /* wwEditor:start */
        wwEditorState: { type: Object, required: true },
        /* wwEditor:end */
    },
    emits: ["trigger-event"],
    setup(props) {
        const { value: isDragging, setValue: setDrag } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: "isDragging",
            type: "boolean",
            defaultValue: false,
            readonly: true,
        });
        // Self-contained fallback for standalone use (no parent wiring collapse via ww-props):
        // not readonly, so it also stays settable via a normal "Set variable" workflow action.
        // When there IS a parent (ww-kanban), the `collapsed` computed below defers to its
        // `collapsed` ww-prop instead, and the "Toggle collapse" element action (see
        // toggleCollapsed/ww-config.js actions) asks the parent to toggle it via
        // customCollapseHandler rather than touching this variable at all. There's no built-in
        // click handler anywhere in this file - toggling is entirely up to whatever the designer
        // wires inside headerElement/collapsedHeaderElement via that action.
        const { value: internalCollapsed, setValue: setInternalCollapsed } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: "collapsed",
            type: "boolean",
            defaultValue: !!props.content.collapsed,
        });
        watch(
            () => props.content.collapsed,
            (value) => setInternalCollapsed(!!value)
        );
        return { isDragging, setDrag, internalCollapsed, setInternalCollapsed };
    },
    data: () => ({
        internalItems: [],
    }),
    methods: {
        toggleCollapsed() {
            if (this.customCollapseHandler) {
                this.customCollapseHandler({ ...this.wwElementState.props });
            } else {
                this.setInternalCollapsed(!this.internalCollapsed);
            }
        },
        onChange(change) {
            this.customHandler &&
                this.customHandler(change, { ...this.wwElementState.props, updatedStackItems: this.internalItems });
            if (change.moved) {
                this.$emit("trigger-event", {
                    name: "item:moved",
                    event: {
                        item: change.moved.element,
                        oldIndex: change.moved.oldIndex,
                        newIndex: change.moved.newIndex,
                        updatedList: this.internalItems,
                    },
                });
            }

            if (change.added) {
                this.$emit("trigger-event", {
                    name: "item:added",
                    event: {
                        item: change.added.element,
                        newIndex: change.added.newIndex,
                        updatedList: this.internalItems,
                    },
                });
            }

            if (change.removed) {
                this.$emit("trigger-event", {
                    name: "item:removed",
                    event: {
                        item: change.removed.element,
                        oldIndex: change.removed.oldIndex,
                        updatedList: this.internalItems,
                    },
                });
            }
        },
    },
    computed: {
        isEditing() {
            /* wwEditor:start */
            return this.wwEditorState.editMode === wwLib.wwEditorHelper.EDIT_MODES.EDITION;
            /* wwEditor:end */
            // eslint-disable-next-line no-unreachable
            return false;
        },
        items() {
            const data = this.wwElementState.props.items ? this.wwElementState.props.items : this.content.items;
            const items = wwLib.wwCollection.getCollectionData(data);
            if (!Array.isArray(items)) return [];
            return items;
        },
        group() {
            return this.wwElementState.props.group ? this.wwElementState.props.group : this.content.group;
        },
        sortable() {
            return this.wwElementState.props.sortable ? this.wwElementState.props.sortable : this.content.sortable;
        },
        itemKey() {
            return this.wwElementState.props.itemKey || "id";
        },
        handle() {
            return this.wwElementState.props.handle?.length
                ? this.wwElementState.props.handle
                : this.content.customDragHandle
                ? this.content.handleClass || "draggable"
                : null;
        },
        isReadonly() {
            /* wwEditor:start */
            if (this.wwEditorState.isSelected) {
                return this.wwElementState.states.includes("readonly");
            }
            /* wwEditor:end */
            // Ensure to return a boolean as vuedraggable interpret undefined as true
            return !!(this.wwElementState.props.readonly || this.content.readonly);
        },
        collapsed() {
            /* wwEditor:start */
            // While selected in the editor, defer to the states list so switching to the
            // "Collapsed" state tab actually previews the collapsed look to design against,
            // rather than whatever the real runtime value happens to be.
            if (this.wwEditorState.isSelected) {
                return this.wwElementState.states.includes("collapsed");
            }
            /* wwEditor:end */
            return this.wwElementState.props.collapsed !== undefined
                ? this.wwElementState.props.collapsed
                : this.internalCollapsed;
        },
        hideHeader() {
            return !!this.wwElementState.props.hideHeader;
        },
        hideFooter() {
            return !!this.wwElementState.props.hideFooter;
        },
    },
    watch: {
        items: {
            immediate: true,
            deep: true,
            handler: function (value) {
                this.internalItems = [...value];
            },
        },
        isDragging(value) {
            if (this.customDragHandler) {
                this.customDragHandler(value, { ...this.wwElementState.props });
            }
        },
        isReadonly: {
            immediate: true,
            handler(value) {
                if (value) {
                    this.$emit("add-state", "readonly");
                } else {
                    this.$emit("remove-state", "readonly");
                }
            },
        },
        collapsed: {
            immediate: true,
            handler(value) {
                if (value) {
                    this.$emit("add-state", "collapsed");
                } else {
                    this.$emit("remove-state", "collapsed");
                }
            },
        },
    },
};
</script>

<style scoped>
/** FIX POINTER-EVENTS: ALL BREAKING DRAGGABLE ON MOBILE/TABLET (TOUCH MODE) */
.draggable-item :deep(.ww-layout) {
    pointer-events: unset !important;
}
.draggable-item :deep(* > .ww-element) {
    pointer-events: unset !important;
}
.draggable-item :deep(* .ww-element) {
    pointer-events: unset !important;
}

/**
 * Collapse hides items/footer with display:none rather than unmounting the draggable root
 * (no v-if anywhere in this file) - the sortable container itself stays exactly as mounted
 * and registered with SortableJS as it was expanded, so it remains a valid drop target while
 * collapsed; only its visible footprint shrinks to whatever the header content renders as.
 */
.ww-stack-draggable--collapsed :deep(.draggable-item),
.ww-stack-draggable--collapsed :deep(.ww-stack-footer) {
    display: none;
}
</style>
