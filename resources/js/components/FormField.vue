<template>
  <component
    :dusk="currentField.attribute"
    :is="currentField.fullWidth ? 'FullWidthField' : 'default-field'"
    :field="currentField"
    :errors="errors"
    :show-help-text="showHelpText"
    full-width-content>
    <template #field>

      <div ref="flexibleFieldContainer">
        <form-nova-flexible-content-group
          v-for="(group, index) in orderedGroups"
          :dusk="currentField.attribute + '-' + index"
          :key="group.key"
          :field="currentField"
          :group="group"
          :index="index"
          :resource-name="resourceName"
          :resource-id="resourceId"
          :errors="errors"
          :mode="mode"
          @move-up="moveUp(group.key)"
          @move-down="moveDown(group.key)"
          @remove="remove(group.key)"
          @field-changed="handleFieldChange"
        />
      </div>

      <component
        :layouts="layouts"
        :is="currentField.menu.component"
        :field="currentField"
        :limit-counter="limitCounter"
        :limit-per-layout-counter="limitPerLayoutCounter"
        :errors="errors"
        :resource-name="resourceName"
        :resource-id="resourceId"
        @addGroup="addGroup($event)"
      />

    </template>
  </component>
</template>

<script>

import FullWidthField from './FullWidthField';
import Sortable from 'sortablejs'
import { DependentFormField, HandlesValidationErrors, mapProps } from 'laravel-nova';
import Group from '../group';

export default {
  mixins: [HandlesValidationErrors, DependentFormField],

  props: {
    ...mapProps(['mode']),
  },

  components: { FullWidthField },

  computed: {
    layouts() {
      return this.currentField.layouts || false
    },
    orderedGroups() {
      return this.order.reduce((groups, key) => {
        groups.push(this.groups[key]);
        return groups;
      }, []);
    },

    limitCounter() {
      if (this.currentField.limit === null || typeof(this.currentField.limit) == "undefined") {
        return null;
      }

      return this.currentField.limit - Object.keys(this.groups).length;
    },

    limitPerLayoutCounter() {
      return this.layouts.reduce((layoutCounts, layout) => {
        if (layout.limit === null) {
          layoutCounts[layout.name] = null;

          return layoutCounts;
        }

        let count = Object.values(this.groups).filter(group => group.name === layout.name).length;

        layoutCounts[layout.name] = layout.limit - count;

        return layoutCounts;
      }, {});
    },
  },

  data() {
    return {
      order: [],
      groups: {},
      files: {},
      sortableInstance: null,
      fieldsDuplication: null,
      changeTimeout: null,
      mutationObserver: null
    };
  },

  beforeUnmount() {
    if (this.sortableInstance) {
      this.sortableInstance.destroy();
    }
    if (this.mutationObserver) {
      this.mutationObserver.disconnect();
    }
    if (this.changeTimeout) {
      clearTimeout(this.changeTimeout);
    }
  },

  watch: {
    // Watch for changes in group values to detect select field changes
    groups: {
      handler() {
        this.$nextTick(() => {
          this.handleFieldChange();
        });
      },
      deep: true
    }
  },

  methods: {
    /*
     * Set the initial, internal value for the field.
     */
    setInitialValue() {
      this.value = this.currentField.value || [];
      this.files = {};

      this.populateGroups();
      this.$nextTick(() => {
        this.initSortable();
        this.addAllFieldWatchers();
        this.initMutationObserver();
      });
    },

    /**
     * Fill the given FormData object with the field's internal value.
     */
    fill(formData) {
      let key, group;

      this.value = [];
      this.files = {};

      for (var i = 0; i < this.order.length; i++) {
        key = this.order[i];
        group = this.groups[key].serialize();

        // Only serialize the group's non-file attributes
        this.value.push({
          layout: group.layout,
          key: group.key,
          attributes: group.attributes
        });

        // Attach the files for formData appending
        this.files = {...this.files, ...group.files};
      }

      this.appendFieldAttribute(formData, this.currentField.attribute);
      formData.append(this.currentField.attribute, this.value.length ? JSON.stringify(this.value) : '');

      // Append file uploads
      for (let file in this.files) {
        formData.append(file, this.files[file]);
      }

      this.$nextTick(this.initSortable.bind(this));
    },

    /**
     * Register given field attribute into the parsable flexible fields register
     */
    appendFieldAttribute(formData, attribute) {
      let registered = [];

      if (formData.has('___nova_flexible_content_fields')) {
        registered = JSON.parse(formData.get('___nova_flexible_content_fields'));
      }

      registered.push(attribute);

      formData.set('___nova_flexible_content_fields', JSON.stringify(registered));
    },

    /**
     * Update the field's internal value.
     */
    handleChange(value) {
      this.value = value || [];
      this.files = {};
      this.populateGroups();
    },

    /**
     * Handle field change events from groups
     */
    handleFieldChange() {
      // Debounce to avoid multiple rapid calls
      if (this.changeTimeout) {
        clearTimeout(this.changeTimeout);
      }

      this.changeTimeout = setTimeout(() => {
        // Rebuild the value array from current groups
        this.value = [];
        this.files = {};

        for (var i = 0; i < this.order.length; i++) {
          const key = this.order[i];
          const group = this.groups[key].serialize();

          // Only serialize the group's non-file attributes
          this.value.push({
            layout: group.layout,
            key: group.key,
            attributes: group.attributes
          });

          // Attach the files for formData appending
          this.files = {...this.files, ...group.files};
        }

        // Emit the change to parent components if needed
        this.emitFieldValueChange(this.currentField.attribute, this.value);
        this.$emit('field-changed');
      }, 1000); // 100ms debounce
    },

    /**
     * Set the displayed layouts from the field's current value
     */
    populateGroups() {
      this.order.splice(0, this.order.length);
      this.groups = {};

      for (var i = 0; i < this.value.length; i++) {
        this.addGroup(
          this.getLayout(this.value[i].layout),
          this.value[i].attributes,
          this.value[i].key,
          this.currentField.collapsed
        );
      }

      // Add watchers after all groups are populated
      this.$nextTick(() => {
        this.addAllFieldWatchers();
        // Reinitialize mutation observer if it was destroyed
        if (!this.mutationObserver) {
          this.initMutationObserver();
        }
      });
    },

    /**
     * Retrieve layout definition from its name
     */
    getLayout(name) {
      if(!this.layouts) return;
      return this.layouts.find(layout => layout.name == name);
    },

    /**
     * Append the given layout to flexible content's list
     */
    addGroup(layout, attributes, key, collapsed) {
      if(!layout) return;

      collapsed = collapsed || false;

      let cleanAttributes = function (attributes) {
        attributes.forEach((element, index, array) => {
          element.attribute = element.attribute.split('__')[1] ?? element.attribute;
          element.validationKey = element.validationKey.split('__')[1] ?? element.validationKey;

          if (element.component === 'nova-flexible-content') {
            element.value.forEach((value, index, array) => {
              array[index].attributes = cleanAttributes(value.attributes);
            });
          }

          array[index] = element;
        });

        return attributes;
      }

      let cleaned_attributes;

      if (attributes)
        cleaned_attributes = cleanAttributes(JSON.parse(JSON.stringify(attributes)));

      if (this.field.duplicateOnCreate) {
        if (attributes && this.fieldsDuplication === null) {
          this.fieldsDuplication = cleanAttributes(JSON.parse(JSON.stringify(attributes)));
        }

        if (attributes === undefined && this.fieldsDuplication !== null) {
          cleaned_attributes = JSON.parse(JSON.stringify(this.fieldsDuplication));
        }
      }

      let fields = cleaned_attributes || JSON.parse(JSON.stringify(layout.fields)),
        group = new Group(layout.name, layout.title, fields, this.currentField, key, collapsed);

      this.groups[group.key] = group;
      this.order.push(group.key);

      // Add watchers for individual field values to catch select changes
      this.$nextTick(() => {
        this.addFieldWatchers(group);
      });
    },

    /**
     * Add watchers for individual field values to detect changes
     */
    addFieldWatchers(group) {
      // Find all form components in this group
      const groupElement = document.getElementById(group.key);
      if (!groupElement) return;

      // Watch for changes in select fields and other form elements
      const formElements = groupElement.querySelectorAll('select, input, textarea');
      formElements.forEach(element => {
        element.addEventListener('change', () => {
          this.$nextTick(() => {
            this.handleFieldChange();
          });
        });
      });
    },

    /**
     * Add watchers for all field values in all groups
     */
    addAllFieldWatchers() {
      // Use the enhanced method to find all form elements
      this.attachListenersToAllFields();
    },

    /**
     * Attach change listener to a form element
     */
    attachFieldListener(element) {
      // Skip if already has listener
      if (element._flexibleListenerAttached) return;

      // Remove existing listeners to avoid duplicates
      element.removeEventListener('change', this.handleFieldChange);
      element.addEventListener('change', () => {
        this.$nextTick(() => {
          this.handleFieldChange();
        });
      });

      // Mark as having listener attached
      element._flexibleListenerAttached = true;
    },

    /**
     * Enhanced method to find and attach listeners to all form elements
     */
    attachListenersToAllFields() {
      const container = this.$refs.flexibleFieldContainer;
      if (!container) return;

      // Find all form elements including those in Nova components
      const selectors = [
        'select',
        'input[type="text"]',
        'input[type="email"]',
        'input[type="number"]',
        'input[type="tel"]',
        'input[type="url"]',
        'input[type="search"]',
        'input[type="password"]',
        'input[type="date"]',
        'input[type="datetime-local"]',
        'input[type="time"]',
        'input[type="checkbox"]',
        'input[type="radio"]',
        'textarea',
        // Nova specific selectors
        '[data-testid*="field"]',
        '.form-control',
        '.nova-field'
      ];

      selectors.forEach(selector => {
        const elements = container.querySelectorAll(selector);
        elements.forEach(element => {
          this.attachFieldListener(element);
        });
      });
    },

    /**
     * Initialize MutationObserver to watch for dynamically added fields
     */
    initMutationObserver() {
      const container = this.$refs.flexibleFieldContainer;
      if (!container || this.mutationObserver) return;

      this.mutationObserver = new MutationObserver((mutations) => {
        let shouldUpdate = false;

        mutations.forEach((mutation) => {
          // Check for added nodes
          if (mutation.type === 'childList') {
            mutation.addedNodes.forEach((node) => {
              if (node.nodeType === Node.ELEMENT_NODE) {
                // Check if the added node or its children contain form elements
                const selectors = [
                  'select', 'input', 'textarea',
                  '[data-testid*="field"]', '.form-control', '.nova-field'
                ];

                selectors.forEach(selector => {
                  const elements = node.querySelectorAll ?
                    node.querySelectorAll(selector) : [];

                  if (elements.length > 0) {
                    elements.forEach(element => {
                      this.attachFieldListener(element);
                    });
                    shouldUpdate = true;
                  }

                  // Also check if the node itself matches the selector
                  if (node.matches && node.matches(selector)) {
                    this.attachFieldListener(node);
                    shouldUpdate = true;
                  }
                });
              }
            });
          }

          // Check for attribute changes that might show/hide fields
          if (mutation.type === 'attributes' &&
            (mutation.attributeName === 'style' || mutation.attributeName === 'class')) {
            const target = mutation.target;
            const selectors = [
              'select', 'input', 'textarea',
              '[data-testid*="field"]', '.form-control', '.nova-field'
            ];

            selectors.forEach(selector => {
              if (target.matches && target.matches(selector)) {
                this.attachFieldListener(target);
                shouldUpdate = true;
              }
            });
          }
        });

        if (shouldUpdate) {
          this.$nextTick(() => {
            this.handleFieldChange();
          });
        }
      });

      // Start observing
      this.mutationObserver.observe(container, {
        childList: true,
        subtree: true,
        attributes: true,
        attributeFilter: ['style', 'class']
      });
    },

    /**
     * Force re-attach listeners to all fields (useful for debugging or manual refresh)
     */
    refreshFieldListeners() {
      this.attachListenersToAllFields();
    },

    /**
     * Move a group up
     */
    moveUp(key) {
      let index = this.order.indexOf(key);

      if(index <= 0) return;

      this.order.splice(index - 1, 0, this.order.splice(index, 1)[0]);
    },

    /**
     * Move a group down
     */
    moveDown(key) {
      let index = this.order.indexOf(key);

      if(index < 0 || index >= this.order.length - 1) return;

      this.order.splice(index + 1, 0, this.order.splice(index, 1)[0]);
    },

    /**
     * Remove a group
     */
    remove(key) {
      let index = this.order.indexOf(key);

      if(index < 0) return;

      this.order.splice(index, 1);
      delete this.groups[key];
    },


    initSortable() {
      const containerRef = this.$refs['flexibleFieldContainer']

      if (! containerRef || this.sortableInstance) {
        return;
      }

      this.sortableInstance = Sortable.create(containerRef, {
        ghostClass: 'nova-flexible-content-sortable-ghost',
        dragClass: 'nova-flexible-content-sortable-drag',
        chosenClass: 'nova-flexible-content-sortable-chosen',
        direction: 'vertical',
        handle: '.nova-flexible-content-drag-button',
        scrollSpeed: 5,
        animation: 500,
        onEnd: (evt) => {
          const item = evt.item;
          const key = item.id;
          const oldIndex = evt.oldIndex;
          const newIndex = evt.newIndex;

          if (newIndex < oldIndex) {
            this.moveUp(key);
          } else if (newIndex > oldIndex) {
            this.moveDown(key);
          }
        }
      });
    },
  }
}
</script>
