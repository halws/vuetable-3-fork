<template>
  <tr>
    <template v-for="(field, fieldIndex) in vuetable.tableFields">
      <template v-if="field.visible">
        <template v-if="vuetable.isFieldComponent(field.name)">
          <component
            :is="field.name"
            :row-field="field"
            :is-header="true"
            :title="renderTitle(field)"
            :vuetable="vuetable"
            :key="fieldIndex"
            :class="headerClass('vuetable-th-component', field)"
            :style="{ width: field.width }"
            @vuetable:header-event="vuetable.onHeaderEvent"
            @click="onColumnHeaderClicked(field, $event)"
          >
            <component
              :is="getSortComponent(field).component"
              v-bind="getSortComponent(field).props"
              v-if="getSortComponent(field)"
            />
          </component>
        </template>
        <template v-else-if="vuetable.isFieldSlot(field.name)">
          <th
            :class="headerClass('vuetable-th-slot', field)"
            :key="fieldIndex"
            :style="{ width: field.width }"
            v-html="renderTitle(field)"
            @click="onColumnHeaderClicked(field, $event)"
          ></th>
        </template>
        <template v-else>
          <th
            @click="onColumnHeaderClicked(field, $event)"
            :key="fieldIndex"
            :id="'_' + field.name"
            :class="headerClass('vuetable-th', field)"
            :style="{ width: field.width }"
            v-html="renderTitle(field)"
          ></th>
        </template>
      </template>
    </template>
    <vuetable-col-gutter v-if="vuetable.scrollVisible" />
  </tr>
</template>
<script>
  /* eslint-disable @typescript-eslint/camelcase */
  /* eslint-disable prefer-const */
  import VuetableFieldCheckbox from "./VuetableFieldCheckbox";
  import VuetableFieldHandle from "./VuetableFieldHandle";
  import VuetableFieldSequence from "./VuetableFieldSequence";
  import VuetableColGutter from "./VuetableColGutter";

  export default {
    components: {
      "vuetable-field-checkbox": VuetableFieldCheckbox,
      "vuetable-field-handle": VuetableFieldHandle,
      "vuetable-field-sequence": VuetableFieldSequence,
      VuetableColGutter,
    },

    computed: {
      sortOrder() {
        return this.$parent.sortOrder;
      },

      css() {
        return this.$parent.$_css;
      },

      vuetable() {
        return this.$parent;
      },
    },

    methods: {
      stripPrefix(name) {
        return name.replace(this.vuetable.fieldPrefix, "");
      },

      headerClass(base, field) {
        return [
          base + "-" + this.toSnakeCase(field.name),
          field.titleClass || "",
          this.sortClass(field),
          { sortable: this.vuetable.isSortable(field) },
        ];
      },

      toSnakeCase(str) {
        return (
          typeof str === "string" &&
          str
            .replace(/([A-Z])/g, (chr) => "_" + chr.toLowerCase())
            .replace(" ", "_")
            .replace(".", "_")
        );
      },

      sortClass(field) {
        let cls = "";
        const i = this.currentSortOrderPosition(field);

        if (i !== false) {
          cls =
            this.sortOrder[i].direction == "asc"
              ? this.css.ascendingClass
              : this.css.descendingClass;
        }

        return cls;
      },

      sortIcon(field) {
        let cls = this.css.sortableIcon;
        const i = this.currentSortOrderPosition(field);

        if (i !== false) {
          cls =
            this.sortOrder[i].direction == "asc"
              ? this.css.ascendingIcon
              : this.css.descendingIcon;
        }

        return cls;
      },

      // Allow ability to render a vue component for the sort icon.
      getSortComponent(field) {
        const i = this.currentSortOrderPosition(field);
        const sortCompFunc = this.css.renderSortComp;
        if (!sortCompFunc) {
          return;
        }

        if (i !== false) {
          return {
            component: sortCompFunc,
            props: {
              order: this.sortOrder[i].direction == "asc" ? "asc" : "desc",
            },
          };
        }

        return {
          component: sortCompFunc,
          props: {
            order: "sortable",
          },
        };
      },

      isInCurrentSortGroup(field) {
        return this.currentSortOrderPosition(field) !== false;
      },

      hasSortableIcon(field) {
        return this.vuetable.isSortable(field) && this.css.sortableIcon != "";
      },

      currentSortOrderPosition(field) {
        if (!this.vuetable.isSortable(field)) {
          return false;
        }

        for (let i = 0; i < this.sortOrder.length; i++) {
          if (this.fieldIsInSortOrderPosition(field, i)) {
            return i;
          }
        }

        return false;
      },

      fieldIsInSortOrderPosition(field, i) {
        return this.sortOrder[i].sortField === field.sortField;
      },
      createElement: (str) => {
        const el = document.createElement("div");
        el.innerHTML = str;
        return el.firstElementChild;
      },
      renderTitle(field) {
        const title = this.getTitle(field);

        if (
          (title.length > 0 && this.isInCurrentSortGroup(field)) ||
          this.hasSortableIcon(field)
        ) {
          const style = `opacity:${this.sortIconOpacity(
            field
          )};position:relative;float:right`;
          const iconTag = this.vuetable.showSortIcons
            ? this.renderIconTag(
                ["sort-icon", this.sortIcon(field)],
                `style="${style}"`,
                field
              )
            : "";

          const template = `<div><span>${title}</span> <span>${iconTag}</span></div`;

          return this.createElement(template);
        }

        return title;
      },

      getTitle(field) {
        if (typeof field.title === "function") return field.title();

        return typeof field.title === "undefined"
          ? field.name.replace(".", " ")
          : field.title;
      },

      sortIconOpacity(field) {
        /*
         * fields with stronger precedence have darker color
         *
         * if there are few fields, we go down by 0.3
         * ex. 2 fields are selected: 1.0, 0.7
         *
         * if there are more we go down evenly on the given spectrum
         * ex. 6 fields are selected: 1.0, 0.86, 0.72, 0.58, 0.44, 0.3
         */
        let max = 1.0,
          min = 0.3,
          step = 0.3;

        const count = this.sortOrder.length;
        const current = this.currentSortOrderPosition(field);

        if (max - count * step < min) {
          step = (max - min) / (count - 1);
        }

        const opacity = max - current * step;

        return opacity;
      },

      renderIconTag(classes, options = "", field) {
        return typeof this.css.renderIcon === "undefined"
          ? `<i class="${classes.join(" ")}" ${options}></i>`
          : this.css.renderIcon(classes, options, field);
      },

      onColumnHeaderClicked(field, event) {
        this.vuetable.orderBy(field, event);
      },
    },
  };
</script>
