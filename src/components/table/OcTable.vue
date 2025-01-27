<template>
  <div>
    <div>
      <div>
        <div
          v-if="groupingSettings && groupingAllowed && groupingSettings.showGroupingOptions"
          class="oc-docs-width-small"
          style="display: inline"
        >
          <label class="oc-mx-s">Group By:</label>
        </div>
        <div
          v-if="groupingSettings && groupingAllowed && groupingSettings.showGroupingOptions"
          class="oc-docs-width-medium"
          style="display: inline-block; width: 250px"
        >
          <oc-select
            v-model="selected"
            :options="['None', ...Object.keys(groupingSettings.groupingFunctions)]"
            :clearable="false"
            :searchable="false"
          />
        </div>
      </div>
      <br />
    </div>
    <table v-bind="extractTableProps()">
      <oc-thead v-if="hasHeader">
        <oc-tr class="oc-table-header-row">
          <oc-th
            v-for="(field, index) in fields"
            :key="`oc-thead-${field.name}`"
            v-bind="extractThProps(field, index)"
            @click.native="clickedField(field)"
          >
            <span
              v-if="field.headerType === 'slot'"
              :key="field.name + 'Header'"
              class="oc-table-thead-content"
            >
              <slot :name="field.name + 'Header'" />
            </span>
            <span
              v-else
              :key="field.name + 'Header'"
              class="oc-table-thead-content"
              v-text="extractFieldTitle(field)"
            />
            <oc-button
              v-if="field.sortable"
              :aria-label="getSortLabel(field.name)"
              class="oc-button-sort"
              variant="passive"
              appearance="raw"
              @click.stop="clickedField(field)"
            />
          </oc-th>
        </oc-tr>
      </oc-thead>
      <oc-tbody v-if="selected === 'None' || !selected">
        <oc-tr
          v-for="(item, trIndex) in tableData"
          :key="`oc-tbody-tr-${item[idKey] || trIndex}`"
          :ref="`row-${trIndex}`"
          v-bind="extractTbodyTrProps(item, trIndex)"
          @click.native="$emit(constants.EVENT_TROW_CLICKED, item)"
          @hook:mounted="$emit(constants.EVENT_TROW_MOUNTED, item, $refs[`row-${trIndex}`][0])"
        >
          <oc-td
            v-for="(field, tdIndex) in fields"
            :key="'oc-tbody-td-' + cellKey(field, tdIndex, item)"
            v-bind="extractTdProps(field, tdIndex)"
          >
            <slot v-if="isFieldTypeSlot(field)" :name="field.name" :item="item" />
            <template v-else-if="isFieldTypeCallback(field)">
              {{ field.callback(item[field.name]) }}
            </template>
            <template v-else>{{ item[field.name] }}</template>
          </oc-td>
        </oc-tr>
      </oc-tbody>

      <!-- Collapsibles -->

      <tbody
        v-for="(item, index) in groupedItems"
        v-else-if="groupingAllowed && selected !== 'None'"
        :key="`${item.name + index}`"
      >
        <oc-tr
          style="
             {
              height: rowHeight + 'px';
            }
          "
          :class="['oc-tbody-tr', 'oc-tbody-tr-accordion']"
          @click.native="toggle(item, index)"
          ><oc-td style="colspan='2'"
            ><oc-avatar
              v-if="selected === 'Share owner'"
              :width="30"
              style="
                 {
                  width: 30px;
                  height: 30px;
                }
              "
              :user-name="item.name"
              :src="item.data[0].owner[0].avatar"
              accessible-label="item.name" /></oc-td
          ><oc-td :colspan="fields.length - 2"> {{ item.name }} </oc-td
          ><oc-td>
            <span
              class="oc-ml-xs oc-icon-l"
              :style="[resultArray[index].open ? { display: 'none' } : {}]"
            >
              <oc-icon
                name="expand_less"
                class="oc-accordion-title-arrow-icon"
                :class="{ rotate: resultArray[index].open }"
                size="large"
              />
            </span>
            <span
              class="oc-ml-xs oc-icon-l"
              :style="[resultArray[index].open ? {} : { display: 'none' }]"
            >
              <oc-icon
                name="expand_more"
                class="oc-accordion-title-arrow-icon"
                :class="{ rotate: resultArray[index].open }"
                size="large"
              /> </span></oc-td
        ></oc-tr>
        <template v-if="resultArray[index].open">
          <oc-tr
            v-for="(item, trIndex) in item.data"
            :key="`oc-tbody-tr-${item[idKey] || trIndex}`"
            :ref="`row-${trIndex}`"
            v-bind="extractTbodyTrProps(item, trIndex)"
            @click.native="$emit(constants.EVENT_TROW_CLICKED, item)"
            @hook:mounted="$emit(constants.EVENT_TROW_MOUNTED, item, $refs[`row-${trIndex}`][0])"
          >
            <oc-td
              v-for="(field, tdIndex) in fields"
              :key="'oc-tbody-td-' + cellKey(field, tdIndex, item)"
              v-bind="extractTdProps(field, tdIndex)"
            >
              <slot v-if="isFieldTypeSlot(field)" :name="field.name" :item="item" />
              <template v-else-if="isFieldTypeCallback(field)">
                {{ field.callback(item[field.name]) }}
              </template>
              <template v-else>{{ item[field.name] }}</template>
            </oc-td>
          </oc-tr>
        </template>
      </tbody>

      <!-- Show more/show less footer for preview enabled -->
      <tbody
        v-if="
          groupingSettings &&
          groupingSettings.previewAmount &&
          data.length > groupingSettings.previewAmount + 1
        "
      >
        <oc-tr
          style="
             {
              height: rowHeight + 'px';
            }
          "
          :class="['oc-tbody-tr', 'preview-settings']"
          @click.native="previewEnabled = !previewEnabled"
        >
          <oc-td
            v-if="previewEnabled"
            key="showMore"
            :colspan="fields.length"
            style="
               {
                text-align: center;
              }
            "
            ><div class="preview-heading">Show more</div> </oc-td
          ><oc-td
            v-else
            key="showLess"
            :colspan="fields.length"
            style="
               {
                text-align: center;
              }
            "
            ><div class="preview-heading">Show less</div>
          </oc-td>
        </oc-tr>
      </tbody>

      <tfoot v-if="$slots.footer" class="oc-table-footer">
        <tr>
          <td :colspan="footerColspan" class="oc-table-footer-cell">
            <!-- @slot Footer of the table -->
            <slot name="footer" />
          </td>
        </tr>
      </tfoot>
    </table>
  </div>
</template>
<script>
import OcThead from "./_OcTableHeader"
import OcTbody from "./_OcTableBody"
import OcTr from "./_OcTableRow"
import OcTh from "./_OcTableCellHead"
import OcTd from "./_OcTableCellData"
import OcButton from "../OcButton"
import OcAvatar from "../avatars/OcAvatar.vue"
import SortMixin from "./mixins/sort"
import { getSizeClass } from "../../utils/sizeClasses"
import OcSelect from "../OcSelect.vue"

import { EVENT_THEAD_CLICKED, EVENT_TROW_CLICKED, EVENT_TROW_MOUNTED } from "./helpers/constants"

/**
 * A table component with dynamic layout and data.
 */
export default {
  name: "OcTable",
  status: "ready",
  release: "2.1.0",
  components: {
    OcThead,
    OcTbody,
    OcTr,
    OcTh,
    OcTd,
    OcButton,
    OcAvatar,
    OcSelect,
  },
  mixins: [SortMixin],
  props: {
    /**
     * Grouping settings for the table. Following settings are possible:<br />
     * -**groupingFunctions**: Object with keys as grouping options names and functions that get a table data row and return a group name for that row. The names of the functions are used as grouping options.<br />
     * -**groupingBy**: must be either one of the keys in groupingFunctions or 'None'. If not set, default grouping will be 'None'.<br />
     * -**ShowGroupingOptions**:  boolean value for showing or hinding the select element with grouping options above the table. <br />
     * -**PreviewAmount**: Integer value that is used to show only the first n data rows of the table.
     */
    groupingSettings: {
      type: Object,
      required: false,
    },
    /**
     * The data for the table. Each array item will be rendered as one table row. Each array item needs to have a
     * unique identifier. By default we expect this to be an `id` field. If your field has a different name, please
     * specify it in the `id-key` property of oc-table.
     */
    data: {
      type: Array,
      required: true,
    },
    /**
     * Name of the id property of your data items. See `data` for details on how to use it. The [idKey] is a required field
     * within your data items if you want to have working highlighting. For data representation it is not needed.
     */
    idKey: {
      type: String,
      default: "id",
    },
    /**
     * The column layout of the table.
     *
     * Each field can have the following data:<br />
     * - **name**: values need to be keys of your data items. Required.<br />
     * - **title**: title as displayed in the table header. Optional, falls back to the value of name.<br />
     * - **headerType**: the header field type, can be `slot`, entirely absent or unknown. If absent or unknown, the data will be rendered into a plain table cell.<br />
     * - **type**: the field type, can be `slot`, `callback`, entirely absent or unknown. If absent or unknown, the data will be rendered into a plain table cell.<br />
     * - **callback**: if `type="callback"` the return value of field.callback will be rendered into a plain table cell.<br />
     * - **alignH**: horizontal cell content alignment, can be `left`, `center` or `right`. Defaults to `left`.<br />
     * - **alignV**: vertical cell content alignment, can be `top`, `middle` or `bottom`. Defaults to `middle`.<br />
     * - **width**: horizontal size of a cell, can be `auto`, `shrink` or `expand`. Defaults to `auto`.<br />
     * - **wrap**: text behaviour of a data cell, can be `truncate`, `overflow`, `nowrap`, `break`. Omitted if not set. Header cells are always fixed to `nowrap`.<br />
     * - **thClass**:additional classes on header cells, provided as a string, classes separated by spaces. Optional, falls back to an empty string.<br />
     * - **tdClass**: additional classes on data cells, provided as a string, classes separated by spaces. Optional, falls back to an empty string.<br />
     * - **sortable**: defines if the column is sortable, can be `true` or `false`.
     */
    fields: {
      type: Array,
      required: true,
    },
    /**
     * Asserts whether the table has a header. The header markup is defined in the `fields` array.
     */
    hasHeader: {
      type: Boolean,
      default: true,
    },
    /**
     * Asserts whether the header of the table is sticky.
     */
    sticky: {
      type: Boolean,
      required: false,
      default: false,
    },
    /**
     * Asserts whether table rows should be highlighted when hovered.
     */
    hover: {
      type: Boolean,
      default: false,
    },
    /**
     * The ids of highlighted data items. Null or an empty string/array for no highlighting.
     */
    highlighted: {
      type: [String, Array],
      default: null,
    },
    /**
     * The ids of disabled data items. Null or an empty string/array for no disabled items.
     */
    disabled: {
      type: [String, Array],
      default: null,
    },
    /**
     * Top position of header used when the header is sticky in pixels
     */
    headerPosition: {
      type: Number,
      required: false,
      default: 0,
    },
    /**
     * Sets the padding size for x axis
     * @values xsmall, small, medium, large, xlarge
     */
    paddingX: {
      type: String,
      required: false,
      default: "small",
      validator: size => /(xsmall|small|medium|large|xlarge)/.test(size),
    },
  },
  data() {
    return {
      selected:
        this.groupingSettings && this.groupingSettings.groupingBy
          ? this.groupingSettings.groupingBy
          : "None",
      accordionClosed: [],
      previewEnabled: true,
      previewData: [],
      resultArray: [],
      showMore: false,
      groupingOrderAsc:
        this.groupingSettings && this.groupingSettings.groupingFunctions
          ? this.groupingOrder()
          : {},
      constants: {
        EVENT_THEAD_CLICKED,
        EVENT_TROW_CLICKED,
        EVENT_TROW_MOUNTED,
      },
    }
  },
  computed: {
    tableData() {
      if (
        this.groupingSettings &&
        this.groupingSettings.previewAmount &&
        this.data.length > this.groupingSettings.previewAmount + 1 &&
        this.selected === "None" &&
        this.previewEnabled
      )
        return this.sortedData
          ? this.sortedData.slice(0, this.groupingSettings.previewAmount || 3)
          : this.data.slice(0, this.groupingSettings.previewAmount || 3)
      return this.sortedData || this.data
    },
    tableClasses() {
      const result = ["oc-table"]

      if (this.hover) {
        result.push("oc-table-hover")
      }

      if (this.sticky) {
        result.push("oc-table-sticky")
      }

      return result
    },

    footerColspan() {
      return this.fields.length
    },
    //my computed
    groupedItems() {
      let result = this.createGroupedItems(
        this.selected,
        this.tableData,
        this.groupingOrderAsc[this.selected]
      )

      if (this.groupingSettings && this.previewEnabled && this.groupingSettings.previewAmount) {
        let previewCount = this.groupingSettings.previewAmount + 1

        result.forEach(e => {
          let eLength = e.data.length
          if (eLength - previewCount >= 0) e.data = e.data.slice(0, previewCount - 1)
          previewCount = previewCount - eLength
        })

        result = result.filter(e => e.data.length > 0)
      }

      this.resultArray = result
      return result
    },
    groupingAllowed() {
      return this.groupingSettings &&
        this.groupingSettings.groupingFunctions &&
        Object.keys(this.groupingSettings.groupingFunctions).length > 0
        ? true
        : false
    },
  },
  methods: {
    //my methods
    groupingOrder() {
      let groupingOrder = {}
      Object.keys(this.groupingSettings.groupingFunctions).forEach(
        group => (groupingOrder[group] = true)
      )
      return groupingOrder
    },
    createGroupedItems(col, data, asc) {
      let groups = {}
      let resultArray = []

      if (Object.keys(this.groupingSettings.groupingFunctions).includes(col)) {
        data.forEach(row => {
          groups[this.groupingSettings.groupingFunctions[col](row)]
            ? groups[this.groupingSettings.groupingFunctions[col](row)].push(row)
            : (groups[this.groupingSettings.groupingFunctions[col](row)] = [row])
        })

        for (const [key, value] of Object.entries(groups)) {
          resultArray.push({
            name: key.toUpperCase(),
            open: true,
            data: value,
          })
        }

        if (col === "creation")
          return resultArray.sort((a, b) =>
            Date.parse(a.data[0].sdate) > Date.parse(b.data[0].sdate) ? -1 : 1
          )
        if (asc) return resultArray.sort((a, b) => (a.name > b.name ? 1 : -1))
        else return resultArray.sort((a, b) => (a.name > b.name ? -1 : 1))
      }
    },

    toggle(item, index) {
      this.resultArray[index].open = !this.resultArray[index].open
    },
    itemOpen(item, index) {
      return this.resultArray[index].open
    },
    clickedField(field) {
      this.$emit(this.constants.EVENT_THEAD_CLICKED, field)

      if (this.groupingSettings && this.groupingAllowed) {
        this.groupingOrderAsc[field.name] = !this.groupingOrderAsc[field.name]
        if (field.name === "name")
          this.groupingOrderAsc.alphabetically = !this.groupingOrderAsc.alphabetically
      }
    },
    onSwitchAdvanced() {
      this.isAdvanced = !this.isAdvanced
    },

    ///
    isFieldTypeSlot(field) {
      return field.type === "slot"
    },
    isFieldTypeCallback(field) {
      return ["callback", "function"].indexOf(field.type) >= 0
    },
    extractFieldTitle(field) {
      if (Object.prototype.hasOwnProperty.call(field, "title")) {
        return field.title
      }
      return field.name
    },
    extractTableProps() {
      return {
        class: this.tableClasses,
      }
    },
    extractThProps(field, index) {
      const props = this.extractCellProps(field)
      props.class = `oc-table-header-cell oc-table-header-cell-${field.name}`
      if (Object.prototype.hasOwnProperty.call(field, "thClass")) {
        props.class += ` ${field.thClass}`
      }
      if (this.sticky) {
        props.style = `top: ${this.headerPosition}px;`
      }

      if (index === 0) {
        props.class += ` oc-pl-${getSizeClass(this.paddingX)} `
      }

      if (index === this.fields.length - 1) {
        props.class += ` oc-pr-${getSizeClass(this.paddingX)}`
      }

      this.extractSortThProps(props, field, index)

      return props
    },
    extractTbodyTrProps(item, index) {
      return {
        class: [
          "oc-tbody-tr",
          `oc-tbody-tr-${item[this.idKey] || index}`,
          this.isHighlighted(item) ? "oc-table-highlighted" : undefined,
          this.isDisabled(item) ? "oc-table-disabled" : undefined,
        ].filter(Boolean),
      }
    },
    extractTdProps(field, index) {
      const props = this.extractCellProps(field, index)
      props.class = `oc-table-data-cell oc-table-data-cell-${field.name}`
      if (Object.prototype.hasOwnProperty.call(field, "tdClass")) {
        props.class += ` ${field.tdClass}`
      }
      if (Object.prototype.hasOwnProperty.call(field, "wrap")) {
        props.wrap = field.wrap
      }

      if (index === 0) {
        props.class += ` oc-pl-${getSizeClass(this.paddingX)} `
      }

      if (index === this.fields.length - 1) {
        props.class += ` oc-pr-${getSizeClass(this.paddingX)}`
      }

      return props
    },
    extractCellProps(field) {
      const result = {}
      if (Object.prototype.hasOwnProperty.call(field, "alignH")) {
        result.alignH = field.alignH
      }
      if (Object.prototype.hasOwnProperty.call(field, "alignV")) {
        result.alignV = field.alignV
      }
      if (Object.prototype.hasOwnProperty.call(field, "width")) {
        result.width = field.width
      }

      return result
    },
    isHighlighted(item) {
      if (!this.highlighted) {
        return false
      }

      if (Array.isArray(this.highlighted)) {
        return this.highlighted.indexOf(item[this.idKey]) > -1
      }

      return this.highlighted === item[this.idKey]
    },

    isDisabled(item) {
      if (!this.disabled) {
        return false
      }

      if (Array.isArray(this.disabled)) {
        return this.disabled.indexOf(item[this.idKey]) > -1
      }

      return this.disabled === item[this.idKey]
    },

    cellKey(field, index, item) {
      const prefix = [item[this.idKey], index + 1].filter(Boolean)

      if (this.isFieldTypeSlot(field)) {
        return [...prefix, field.name].join("-")
      }

      if (this.isFieldTypeCallback(field)) {
        return [...prefix, field.callback(item[field.name])].join("-")
      }

      return [...prefix, item[field.name]].join("-")
    },

    getSortLabel(name) {
      const label = this.$gettext("Sort by %{ name }")

      return this.$gettextInterpolate(label, { name })
    },
  },
}
</script>

<style lang="scss">
.preview-heading {
  display: flex;
  align-items: center;
  justify-content: center;
}
.preview-settings {
  background-color: rgb(247, 245, 245);
  cursor: pointer;
}
.grouping-settings {
  display: flex;
  flex-direction: row;
  align-items: center;
}
.oc-tbody-tr-accordion {
  background-color: var(--oc-color-input-bg);
}
.oc-table {
  border-collapse: collapse;
  border-spacing: 0;
  color: var(--oc-color-text-default);
  width: 100%;

  &-hover tr {
    transition: background-color $transition-duration-short ease-in-out;
  }

  tr:not(&-header-row) {
    height: var(--oc-size-height-table-row);
  }

  tr + tr {
    border-top: 1px solid var(--oc-color-border);
  }

  &-hover tr:hover {
    background-color: var(--oc-color-input-border);
  }

  &-highlighted {
    background-color: var(--oc-color-background-highlight);
  }

  &-disabled {
    background-color: var(--oc-color-background-muted);
    opacity: 0.8;
    pointer-events: none;
  }

  &-sticky {
    position: relative;

    .oc-table-header-cell {
      background-color: var(--oc-color-background-default);
      position: sticky;
      z-index: 1;
    }
  }

  &-thead-content {
    vertical-align: middle;
  }

  &-footer {
    border-top: 1px solid var(--oc-color-border);

    &-cell {
      color: var(--oc-color-text-muted);
      font-size: 0.875rem;
      line-height: 1.4;
      padding: var(--oc-space-xsmall);
    }
  }
}
</style>
<docs>
```js
<template>
  <section>
    <h3 class="oc-heading-divider">
      A simple table with plain field types
    </h3>
    <oc-table :fields="fields" :data="data" highlighted="4b136c0a-5057-11eb-ac70-eba264112003"
      disabled="8468c9f0-5057-11eb-924b-934c6fd827a2" :sticky="true">
      <template #footer>
        3 resources
      </template>
    </oc-table>
  </section>
</template>
<script>
  export default {
    computed: {
      fields() {
        return [{
          name: "resource",
          title: "Resource",
          alignH: "left"
        }, {
          name: "last_modified",
          title: "Last modified",
          alignH: "right"
        }]
      },
      data() {
        return [{
          id: "4b136c0a-5057-11eb-ac70-eba264112003",
          resource: "hello-world.txt",
          last_modified: 1609962211
        }, {
          id: "8468c9f0-5057-11eb-924b-934c6fd827a2",
          resource: "I am a folder",
          last_modified: 1608887766
        }, {
          id: "9c4cf97e-5057-11eb-8044-b3d5df9caa21",
          resource: "this is fine.png",
          last_modified: 1599999999
        }]
      }
    }
  }
</script>
```
```js
<template>
  <section>
    <h3 class="oc-heading-divider">
      A simple table with all existing field types
    </h3>
    <oc-table :fields="fields" :data="data">
      <template v-slot:resourceHeader>
        <div class="uk-flex uk-flex-middle">
          <oc-icon name="folder" class="oc-mr-s" />
          Resource
        </div>
      </template>
      <template v-slot:resource="rowData">
        <oc-tag>
          <oc-icon :name="rowData.item.icon" />
          {{ rowData.item.resource }}
        </oc-tag>
      </template>
    </oc-table>
  </section>
</template>
<script>
  export default {
    computed: {
      fields() {
        return [{
          name: "resource",
          title: "Resource",
          headerType: "slot",
          type: "slot"
        }, {
          name: "last_modified",
          title: "Last modified",
          type: "callback",
          callback: function(timestamp) {
            const date = new Date(timestamp * 1000)
            const hours = date.getHours()
            const minutes = "0" + date.getMinutes()
            const seconds = "0" + date.getSeconds()
            return hours + ":" + minutes.substr(-2) + ":" + seconds.substr(-2)
          }
        }]
      },
      data() {
        return [{
          id: "4b136c0a-5057-11eb-ac70-eba264112003",
          resource: "hello-world.txt",
          icon: "text",
          last_modified: 1609962211
        }, {
          id: "8468c9f0-5057-11eb-924b-934c6fd827a2",
          resource: "I am a folder",
          icon: "folder",
          last_modified: 1608887766
        }, {
          id: "9c4cf97e-5057-11eb-8044-b3d5df9caa21",
          resource: "this is fine.png",
          icon: "image",
          last_modified: 1599999999
        }]
      }
    }
  }
</script>
```
```js
<template>
  <section>
    <h3 class="oc-heading-divider">
      A table with long text showing the different text wrapping mechanisms
    </h3>
    <oc-table :fields="fields" :data="data" :has-header="true" :hover="true" />
  </section>
</template>
<script>
  export default {
    computed: {
      fields() {
        return [
          {
            name: "truncate",
            title: "truncate",
            wrap: "truncate"
          },
          {
            name: "break",
            title: "break",
            wrap: "break"
          },
          {
            name: "nowrap",
            title: "nowrap",
            wrap: "nowrap"
          }
        ]
      },
      data() {
        return [
          {
            truncate: "This is very long text that will get truncated eventually. This is very long text that will get truncated eventually. This is very long text that will get truncated eventually. This is very long text that will get truncated eventually. This is very long text that will get truncated eventually. This is very long text that will get truncated eventually. This is very long text that will get truncated eventually.",
            break: "This text is supposed to break to new lines if it becomes too long. This text is supposed to break to new lines if it becomes too long. This text is supposed to break to new lines if it becomes too long. This text is supposed to break to new lines if it becomes too long.",
            nowrap: "This text stays on one line."
          }
        ]
      }
    }
  }
</script>
```
```js
<template>
  <section>
    <h3 class="oc-heading-divider">
      An interactive table showcasing different table features/properties
    </h3>
    <oc-table :fields="fields" :data="data" :has-header="hasHeader" :sticky="stickyHeader" :hover="hover">
      <template v-slot:action="rowData">
        <oc-button @click="toggle(rowData)" size="small">Toggle</oc-button>
      </template>
    </oc-table>
  </section>
</template>
<script>
  export default {
    data() {
      return {
        hasHeader: true,
        stickyHeader: false,
        hover: true
      }
    },
    computed: {
      fields() {
        return [
          {
            name: "property",
            title: "Property",
            sortable: true
          },
          {
            name: "description",
            title: "Description",
            width: "expand",
            sortable: true
          },
          {
            name: "state",
            title: "State",
            width: "shrink",
            sortable: true
          },
          {
            name: "action",
            title: "",
            type: "slot",
            width: "shrink"
          }
        ]
      },
      data() {
        return [
          {
            property: "has-header",
            description: "Whether or not the table header is visible",
            state: this.hasHeader,
            variable: "hasHeader"
          },
          {
            property: "sticky",
            description: "Whether or not the table header is sticky, causing it to float above the table content when scrolling",
            state: this.stickyHeader,
            variable: "stickyHeader"
          },
          {
            property: "hover",
            description: "Highlight table rows on mouseover",
            state: this.hover,
            variable: "hover"
          }
        ]
      }
    },
    methods: {
      toggle(rowData) {
        this[rowData.item.variable] = !this[rowData.item.variable];
      }
    },
  }
</script>
```
</docs>
