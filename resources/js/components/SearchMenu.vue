<template>
    <div class="w-3/5" v-if="layouts">
        <div v-if="field.readonly || this.limitCounter > 0 || this.limitCounter === null">
            <div v-if="layouts.length === 1 || field.readonly">
                <span
                    class="inline-block"
                    :class="{ 'nova-flexible-add-button-disabled': field.readonly }"
                >
                    <default-button
                        dusk="toggle-layouts-dropdown-or-add-default"
                        type="button"
                        :tabindex="field.readonly ? -1 : 0"
                        class="nova-flexible-add-button"
                        :class="{ 'is-readonly': field.readonly }"
                        :disabled="field.readonly"
                        :aria-disabled="field.readonly ? 'true' : 'false'"
                        @click="toggleLayoutsDropdownOrAddDefault"
                    >
                        <span>{{ field.button }}</span>
                    </default-button>
                </span>
            </div>
            <div v-else-if="layouts.length > 1">
                <div style="min-width: 300px;">
                    <div class="flexible-search-menu-multiselect">
                        <Multiselect
                             v-model="selectedLayout"
                             :options="availableLayouts"
                             :placeholder="field.button"
                             @change="selectLayout"
                             v-bind="attributes"
                             track-by="name"
                             :show-options="true"
                             :searchable="true"
                             ref="select"
                        ></Multiselect>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import Multiselect from '@vueform/multiselect'

    export default {
        props: ['layouts', 'field', 'resourceName', 'resourceId', 'resource', 'errors', 'limitCounter', 'limitPerLayoutCounter'],

        emits: ['addGroup'],

        components: {
            Multiselect,
        },

        data() {
            return {
                selectedLayout: null,
                isLayoutsDropdownOpen: false
            };
        },

        computed: {
            attributes() {
                return {
                    selectLabel: this.field.menu.data.selectLabel || this.__('Press enter to select'),
                    label: this.field.menu.data.label || 'title',
                    openDirection: this.field.menu.data.openDirection || 'bottom',
                }
            },

            availableLayouts() {
                return this.layouts.filter(layout => {
                    return this.limitPerLayoutCounter[layout.name] === null || this.limitPerLayoutCounter[layout.name] > 0
                }).reduce((carry, layout) => {
                    carry[layout.name] = layout.title;

                    return carry;
                }, {});
            },
        },

        methods: {
            selectLayout(layoutName){
                if (this.field.readonly) return;

                let layout = this.layouts.find(layout => layout.name === layoutName);
                this.addGroup(layout);
            },

            /**
             * Display or hide the layouts choice dropdown if there are multiple layouts
             * or directly add the only available layout.
             */
            toggleLayoutsDropdownOrAddDefault(event) {
                if (this.field.readonly) {
                    return;
                }

                if (this.layouts.length === 1) {
                    return this.addGroup(this.layouts[0]);
                }

                this.isLayoutsDropdownOpen = !this.isLayoutsDropdownOpen;
            },

            /**
             * Append the given layout to flexible content's list
             */
            addGroup(layout) {
                if (!layout || this.field.readonly) return;

                this.$emit('addGroup', layout);

                this.isLayoutsDropdownOpen = false;

                setTimeout(() => {
                    this.$refs.select.clear();
                    this.selectedLayout = null;
                }, 100);
            },
        }
    }
</script>

<style lang="scss">
.flexible-search-menu-multiselect {
  @import "@vueform/multiselect/themes/default.scss";
}
</style>
