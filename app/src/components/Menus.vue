<template>
    <section :class="{ 'content': true, 'menu': true, 'menus-list-view': true, 'no-scroll': editorVisible }">
        <p-header
            v-if="!showEmptyState"
            :title="$t('menu.menu')">
            <p-button
                :onClick="showAddMenuForm"
                slot="buttons"
                type="primary icon"
                icon="add-site-mono">
                {{ $t('menu.addNewMenu') }}
            </p-button>
        </p-header>

        <collection
            v-if="!showEmptyState"
            :itemsCount="4">
            <collection-header slot="header">
                <collection-cell>
                    <checkbox
                        value="all"
                        :checked="anyCheckboxIsSelected"
                        :onClick="toggleAllCheckboxes.bind(this, true)" />
                </collection-cell>

                <collection-cell>
                    {{ $t('ui.name') }}
                </collection-cell>

                <collection-cell min-width="180px">
                    {{ $t('menu.assignedMenu') }}
                </collection-cell>

                <collection-cell
                    justifyContent="center"
                    textAlign="center"
                    min-width="80px">
                    {{ $t('menu.items') }}
                </collection-cell>

                <div
                    v-if="anyCheckboxIsSelected"
                    class="tools">
                    <p-button
                        icon="trash"
                        type="small light icon"
                        :onClick="bulkDelete">
                        {{ $t('ui.delete') }}
                    </p-button>
                </div>
            </collection-header>

            <collection-row
                v-for="(item, index) in items"
                slot="content"
                :key="index">
                <collection-cell>
                    <checkbox
                        :value="index"
                        :checked="isChecked(index)"
                        :onClick="toggleSelection" />
                </collection-cell>

                <collection-cell>
                    <a
                        href="#"
                        @click.prevent.stop="toggleMenu(index)">
                        {{ item.name }}
                    </a>
                </collection-cell>

                <collection-cell
                    type="assignment">
                    <dropdown
                        :id="'menu-select-' + index"
                        :items="availableMenus"
                        :selected="item.position"
                        :noBorder="true"
                        :disabledValues="usedMenus"
                        @change.native="changeMenu($event, index)">
                    </dropdown>
                </collection-cell>

                <collection-cell
                    justifyContent="center"
                    textAlign="center">
                    <a
                        @click.prevent.stop="toggleMenu(index)"
                        href="#">
                        {{ countMenuItems(item.items) }}
                    </a>
                </collection-cell>

                <div
                    v-if="menuIsOpened(index)"
                    class="item-content">
                    <p-button
                        icon="add-site-mono"
                        type="secondary icon"
                        @click.native="addMenuItem(index)">
                        {{ $t('menu.addMenuItem') }}
                    </p-button>
                    <p-button
                     icon="edit"
                        type="clean icon"
                        class="menu-edit-btn"
                        @click.native="editMenuName(item.name, index)">
                        {{ $t('menu.editMenuName') }}
                    </p-button>

                    <div class="menu-content">
                        <em
                            v-if="!item.items.length"
                            class="menu-empty-list">
                            {{ $t('menu.noMenusMessage') }}
                        </em>

                        <draggable
                            v-if="item.items.length"
                            tag="ol"
                            group="menu-items"
                            chosenClass="is-chosen"
                            ghostClass="is-ghost"
                            class="menu-item-list"
                            v-model="$store.state.currentSite.menuStructure[index].items"
                            v-bind="{
                                animation: 0,
                                forceFallback: true,
                                disabled: !!selectedItem
                            }"
                            :key="'draggable-menu-items-levle0-' + index"
                            @update="listItemUpdated($event, index)"
                            @add="listItemAdded($event, index)">
                            <menu-item
                                v-for="(subitem, subindex) in item.items"
                                :key="'menu-' + subindex + '-' + getUID()"
                                :itemData="subitem"
                                :itemMenuID="index"
                                :itemOrder="subindex"
                                :editedID="editedID"
                                :selectedItem="selectedItemMenuID === index ? selectedItem : null" />
                        </draggable>
                    </div>
                </div>
            </collection-row>
        </collection>

        <empty-state
            v-if="showEmptyState"
            imageName="menus.svg"
            imageWidth="344"
            imageHeight="286"
            :title="$t('menu.noMenusAvailable')"
            :description="$t('menu.noMenusCreateNewOne')">
            <p-button
                slot="button"
                icon="add-site-mono"
                type="icon"
                :onClick="showAddMenuForm">
                {{ $t('menu.addNewMenu') }}
            </p-button>
        </empty-state>

        <transition>
            <menu-item-editor v-if="editorVisible" />
        </transition>
    </section>
</template>

<script>
import Draggable from 'vuedraggable';
import MenuItem from './MenuItem.vue';
import MenuItemEditor from './MenuItemEditor.vue';
import CollectionCheckboxes from './mixins/CollectionCheckboxes.js';

export default {
    name: 'menus',
    mixins: [
        CollectionCheckboxes
    ],
    components: {
        'draggable': Draggable,
        'menu-item': MenuItem,
        'menu-item-editor': MenuItemEditor
    },
    data () {
        return {
            editedID: false,
            editorVisible: false,
            filterValue: '',
            selectedItems: [],
            openedItems: [],
            openedEditForms: [],
            selectedItem: null,
            selectedItemMenuID: null
        };
    },
    computed: {
        items () {
            return this.$store.state.currentSite.menuStructure;
        },
        showEmptyState: function() {
            return !this.items.length;
        },
        availableMenus () {
            let menus = JSON.parse(JSON.stringify(this.$store.state.currentSite.themeSettings.menus));
            menus[''] = this.$t('menu.unassigned');
            return menus;
        },
        usedMenus () {
            return this.items.map(item => item.position);
        }
    },
    mounted () {
        this.$bus.$on('hide-menu-item-editor', () => {
            this.editedID = false;
            this.editorVisible = false;
        });

        this.$bus.$on('show-menu-item-editor-from-submenu', itemID => {
            this.editedID = itemID;
            this.editorVisible = true;
        });

        this.$bus.$on('save-new-menu-structure', () => {
            this.saveNewMenuStructure();
        });

        this.$bus.$on('menus-manager-selected-item', itemID => {
            this.selectMenuItem(itemID);
        });

        this.$bus.$on('menus-manager-unselect-item', () => {
            this.unselectMenuItem();
        });

        this.$bus.$on('menus-manager-move-item', config => {
            this.moveMenuItem(config);
        });
    },
    methods: {
        toggleMenu (index) {
            if (this.openedItems.indexOf(index) === -1) {
                this.openedItems.push(index);
            } else {
                let position = this.openedItems.indexOf(index);
                this.openedItems.splice(position, 1);
            }
        },
        showAddMenuForm () {
            this.$bus.$emit('confirm-display', {
                hasInput: true,
                message: this.$t('menu.provideNameForNewMenu'),
                okClick: (result) => this.addNewMenu(result),
                okLabel: this.$t('menu.createNewMenu')
            });
        },
        addNewMenu (newMenuName) {
            if(newMenuName !== '') {
                if(this.menuNameIsUnique(newMenuName)) {
                    this.$store.commit('addNewMenu', newMenuName);
                    this.saveNewMenuStructure();

                    this.$bus.$emit('message-display', {
                        message: this.$t('menu.newMenuCreated'),
                        type: 'success',
                        lifeTime: 3
                    });
                } else {
                    this.$bus.$emit('message-display', {
                        message: this.$t('menu.menuNameExistsErrorMessage'),
                        type: 'warning',
                        lifeTime: 3
                    });
                }
            } else {
                this.$bus.$emit('message-display', {
                    message: this.$t('menu.menuNameCannotBeEmptyCreateNewMenuErrorMessage'),
                    type: 'warning',
                    lifeTime: 3
                });
            }
        },
        menuNameIsUnique(name, oldName = false) {
            for(let i = 0; i < this.$store.state.currentSite.menuStructure.length; i++) {
                if(oldName) {
                    if(name === this.$store.state.currentSite.menuStructure[i].name && name !== oldName) {
                        return false;
                    }
                } else if(name === this.$store.state.currentSite.menuStructure[i].name) {
                    return false;
                }
            }

            return true;
        },
        changeMenu (event, menuIndex) {
            let newValue = event.target.value;

            this.$store.commit('setMenuPosition', {
                index: menuIndex,
                position: newValue
            });

            this.saveNewMenuStructure();
        },
        menuIsOpened (index) {
            return this.openedItems.indexOf(index) > -1;
        },
        addMenuItem (index) {
            this.editorVisible = true;

            setTimeout(() => {
                this.$bus.$emit('show-menu-item-editor', {
                    menuID: index
                });
            }, 0);
        },
        editMenuName (oldName, index) {
            this.$bus.$emit('confirm-display', {
                hasInput: true,
                message: this.$t('menu.provideNewNameForMenu'),
                okClick: (result) => this.changeMenuName(result, index),
                okLabel: this.$t('menu.editMenuName'),
                defaultText: oldName
            });
        },
        changeMenuName (newMenuName, index) {
            if(newMenuName !== '') {
                if(this.menuNameIsUnique(newMenuName)) {
                    this.$store.commit('editMenuName', {
                        newName: newMenuName,
                        index: index
                    });
                    this.saveNewMenuStructure();

                    this.$bus.$emit('message-display', {
                        message: this.$t('menu.menuNameHasBeenEdited'),
                        type: 'success',
                        lifeTime: 3
                    });
                } else {
                    this.$bus.$emit('message-display', {
                        message: this.$t('menu.menuNameInUseErrorMessage'),
                        type: 'warning',
                        lifeTime: 3
                    });
                }
            } else {
                this.$bus.$emit('message-display', {
                    message: this.$t('menu.menuNameCannotBeEmptyErrorMessage'),
                    type: 'warning',
                    lifeTime: 3
                });
            }
        },
        bulkDelete () {
            this.$bus.$emit('confirm-display', {
                message: this.$t('menu.menusRemoveMessage'),
                okClick: this.deleteSelected
            });
        },
        deleteSelected () {
            let itemsToRemove = this.getSelectedItems(false);

            this.$store.commit('deleteMenuByIDs', itemsToRemove);
            this.saveNewMenuStructure();
            this.selectedItems = [];

            this.$bus.$emit('message-display', {
                message: this.$t('menu.menusRemoveSuccessMessage'),
                type: 'success',
                lifeTime: 3
            });
        },
        saveNewMenuStructure () {
            mainProcessAPI.send('app-menu-update', {
                siteName: this.$store.state.currentSite.config.name,
                menuStructure: this.$store.state.currentSite.menuStructure
            });
        },
        countMenuItems(items) {
            let result = items.length;

            for(let i = 0; i < items.length; i++) {
                if(items[i].items.length) {
                    result += this.countMenuItems(items[i].items);
                }
            }

            return result;
        },
        getUID () {
            return Math.random().toString(36).substr(2, 9);
        },
        listItemUpdated (e, menuID) {
            this.saveNewMenuStructure();
        },
        listItemAdded (e) {
            this.saveNewMenuStructure();
        },
        selectMenuItem (data) {
            this.selectedItem = data.id;
            this.selectedItemMenuID = data.menuID;
        },
        unselectMenuItem () {
            this.selectedItem = null;
            this.selectedItemMenuID = null;
        },
        moveMenuItem (config) {
            if (!this.selectedItem) {
                return;
            }

            this.$store.commit('moveMenuItem', {
                position: config.position,
                menuID: config.menuID,
                targetID: this.selectedItem,
                destinationID: config.destinationID
            });

            this.saveNewMenuStructure();
            this.unselectMenuItem();
        }
    },
    beforeDestroy () {
        this.$bus.$off('hide-menu-item-editor');
        this.$bus.$off('show-menu-item-editor-from-submenu');
        this.$bus.$off('save-new-menu-structure');
        this.$bus.$off('menus-manager-selected-item');
        this.$bus.$off('menus-manager-move-item');
    }
}
</script>

<style lang="scss" scoped>
@import '../scss/variables.scss';

.menu {
    overflow: auto;
    overflow-x: hidden!important;

    &.no-scroll {
        overflow: hidden;
    }

    select {
        width: 15rem;
    }

    .item-content {
        border-bottom: 1px solid var(--border-light-color);
        grid-column-start: 1;
        grid-column-end: 5;
        overflow: hidden;
        padding: 3rem 0 3rem 3rem;
        user-select: none;
        width: 100%;
    }

    .col.assignment {
        padding-top: 0;
        padding-bottom: 0;
    }

    &-content {
        margin: 2.5rem 0 0 0;
    }

    &-item {

        &-list {
            list-style-type: none;
            margin: .25 * $spacing 0;
            padding: 0;
        }
    }

    &-edit-btn {
         color: var(--link-primary-color) !important;

         &:active,
         &:focus,
         &:hover {
            color: var(--link-primary-color-hover) !important;
         }
    }

    &-empty-list {
        display: block;
        text-align: center;
    }
}
</style>
