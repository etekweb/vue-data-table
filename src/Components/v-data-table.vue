<template>
    <table v-if="paginatedData.length == 0">
        <caption>
            <slot name="caption">&nbsp;</slot>
        </caption>
        <thead>
        <tr>
            <th v-for="key in columns">
                {{ key | getDisplayName(displayNames) }}

            </th>
        </tr>
        </thead>
        <tbody>
        <tr>
            <td :colspan="columns.length">
                <slot name="nodata">
                </slot>
            </td>
        </tr>
        </tbody>
    </table>
    <table v-else-if="$scopedSlots.child">
        <caption>
            <slot name="caption">&nbsp;</slot>
        </caption>
        <thead>
        <tr>
            <th v-for="key in columns" @click="sortBy(key)" :class="{ active: sortKey == key }">
                {{ key | getDisplayName(displayNames) }}
          <span v-if="colSortable(key)" class="arrow" :class="sortOrders[key] > 0 ? 'asc' : 'dsc'">
          </span>
            </th>
        </tr>
        </thead>
        <tbody v-for="(entry, index) in paginatedData" @dblclick="toggleChild(index)">
        <tr>
            <td v-for="key in columns">
                <slot :name="key" :entry="entry">
                    {{entry[key]}}
                </slot>
            </td>
        </tr>
        <transition :name="childTransitionClass">
            <tr v-if="$scopedSlots.child && childShow[index]">
                <td :colspan="columns.length">
                    <slot name="child" :entry="entry"></slot>
                </td>
            </tr>
        </transition>
        </tbody>
        <tfoot v-if="itemsPerPage != -1 && this.filteredData.length > this.itemsPerPage">
            {{ currentPage * itemsPerPage - itemsPerPage + 1 }} - {{ Math.min(currentPage * itemsPerPage,  filteredData.length) }} of {{ filteredData.length }}
            <span class="previousPage" @click="previousPage"></span> &nbsp;
            <span class="nextPage" @click="nextPage"></span>
        </tfoot>
    </table>
    <table v-else>
        <caption>
            <slot name="caption">&nbsp;</slot>
        </caption>
        <thead>
        <tr>
            <th v-for="key in columns" @click="sortBy(key)" :class="{ active: sortKey == key }">
                {{ key | getDisplayName(displayNames) }}
          <span v-if="colSortable(key)" class="arrow" :class="sortOrders[key] > 0 ? 'asc' : 'dsc'">
          </span>
            </th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="(entry, index) in paginatedData">
            <td v-for="key in columns">
                <slot :name="key" :entry="entry">
                    {{entry[key]}}
                </slot>
            </td>
        </tr>
        </tbody>
        <tfoot v-if="itemsPerPage != -1 && this.filteredData.length > this.itemsPerPage">
            {{ currentPage * itemsPerPage - itemsPerPage + 1 }} - {{ Math.min(currentPage * itemsPerPage,  filteredData.length) }} of {{ filteredData.length }}
            <span class="previousPage" @click="previousPage"></span> &nbsp;
            <span class="nextPage" @click="nextPage"></span>
        </tfoot>
    </table>
</template>
<script>
    export default {
        name: 'data-table',
        props: {
            data: Array,
            sortedItem: String,
            columnsToDisplay: {
                type: Array,
                default() {
                    return [];
                }
            },
            columnsToNotDisplay: {
                type: Array,
                default: function () {
                    return [];
                }
            },
            columnsToSort: {
                type: Array,
                default() {
                    return [];
                }
            },
            columnsToNotSort: {
                type: Array,
                default() {
                    return [];
                }
            },
            aggregateColumns: {
                type: Boolean,
                default: false
            },
            displayNames: {
                type: Object,
                default() {
                    return {};
                }
            },
            filterKey: {
                type: String,
                default: ""
            },
            childHideable: {
                type: Boolean,
                default: false
            },
            childInitHide: {
                type: Boolean,
                default: false
            },
            childTransitionClass: {
                type: String,
                default: ""
            },
            itemsPerPage: {
                type: Number,
                default: -1
            }
        },
        data() {
            const sortOrders = {};
            this.getColumns(this.columnsToDisplay, this.data, this.columnsToNotDisplay).forEach(key => sortOrders[key] = 1);

            const childShow = [];
            if (this.childInitHide) {
                this.data.forEach(entry => childShow.push(false));
            } else {
                this.data.forEach(entry => childShow.push(true));
            }

            return {
                sortKey: '',
                sortOrders: sortOrders,
                childShow: childShow,
                currentPage: 1
            }
        },
        computed: {
            filteredData() {
                const sortKey = this.sortKey;
                const filterKey = this.filterKey && this.filterKey.toLowerCase();
                const order = this.sortOrders[sortKey] || 1;
                let data = this.data;
                if (filterKey) {
                    data = data.filter(row => {
                        return Object.keys(row).some(key => {
                            return String(row[key]).toLowerCase().indexOf(filterKey) > -1
                        })
                    })
                }
                if (sortKey) {
                    data = data.slice().sort((a, b) => {
                        a = a[sortKey];
                        b = b[sortKey];
                        return (a === b ? 0 : a > b ? 1 : -1) * order
                    })
                }

                return data
            },
            columns() {
                return this.getColumns(this.columnsToDisplay, this.data, this.columnsToNotDisplay);
            },
            totalPages() {
                if (this.itemsPerPage != -1) {
                    return Math.ceil(this.filteredData.length / this.itemsPerPage);
                } else {
                    return 1;
                }
            },
            paginatedData() {
                if (this.itemsPerPage != -1 && this.filteredData.length > this.itemsPerPage) {
                    const index = this.currentPage * this.itemsPerPage - this.itemsPerPage;
                    return this.filteredData.slice(index, index + this.itemsPerPage);
                } else {
                    return this.filteredData;
                }
            }
        },
        filters: {
            getDisplayName(column, displayNames) {
                if (column in displayNames) {
                    return displayNames[column];
                } else {
                    return column.charAt(0).toUpperCase() + column.slice(1);
                }
            }
        },
        methods: {
            sortBy(key) {
                if (this.colSortable(key)) {
                    if (this.childHideable) {
                        this.childShow = this.childShow.map(entry => !this.childInitHide);
                    }
                    this.sortKey = key;
                    this.sortOrders[key] = this.sortOrders[key] * -1
                }
            },

            getColumns(columns, data, columnsToRemove) {
                if (columns.length === 0) {
                    let foundColumns;
                    if (this.aggregateColumns) {
                        let allColumns = [];
                        data.forEach(entry => allColumns = allColumns.concat(Object.keys(entry)));
                        foundColumns = allColumns.filter((item, pos) => allColumns.indexOf(item) == pos);
                    } else {
                        foundColumns = Object.keys(data[0]);
                    }

                    if (columnsToRemove.length > 0) {
                        columnsToRemove.forEach(col => {
                            const index = foundColumns.indexOf(col);
                            if (index > -1) {
                                foundColumns.splice(index, 1);
                            }
                        })
                    }

                    return foundColumns;
                } else {
                    return columns;
                }
            },

            toggleChild(index) {
                if (this.childHideable) {
                    this.childShow[index] = !this.childShow[index];
                    this.childShow.splice(index, 1, this.childShow[index]);
                }
            },

            colSortable(column) {
                if (this.columnsToNotSort.length > 0) {
                    return this.columnsToNotSort.indexOf(column) == -1;
                } else if (this.columnsToSort.length > 0) {
                    return this.columnsToSort.indexOf(column) != -1;
                } else {
                    return true;
                }
            },

            nextPage() {
                if (this.currentPage != this.totalPages) {
                    this.currentPage++;
                }
            },

            previousPage() {
                if (this.currentPage != 1) {
                    this.currentPage--;
                }
            }

        },
        watch: {
            sortedItem: function(){
                this.sortBy(this.sortedItem);
            }
        }
    }
</script>

<style>
</style>
