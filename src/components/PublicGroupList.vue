<template>
    <!-- Search and Filter Bar -->
    <div v-if="!editMode" class="search-filter-bar mb-4">
        <div class="row">
            <div class="col-md-6">
                <div class="input-group">
                    <span class="input-group-text">
                        <font-awesome-icon icon="search" />
                    </span>
                    <input 
                        v-model="searchQuery" 
                        type="text" 
                        class="form-control" 
                        :placeholder="$t('Search monitors...')"
                        @input="filterMonitors"
                    />
                </div>
            </div>
            <div class="col-md-3">
                <select v-model="statusFilter" class="form-select" @change="filterMonitors">
                    <option value="all">{{ $t("All Status") }}</option>
                    <option value="up">{{ $t("Up") }}</option>
                    <option value="down">{{ $t("Down") }}</option>
                    <option value="maintenance">{{ $t("Maintenance") }}</option>
                </select>
            </div>
            <div class="col-md-3">
                <button 
                    v-if="hasActiveFilters" 
                    class="btn btn-outline-secondary w-100" 
                    @click="clearFilters"
                >
                    <font-awesome-icon icon="times" />
                    {{ $t("Clear Filters") }}
                </button>
            </div>
        </div>
        
        <!-- Results counter -->
        <div v-if="hasActiveFilters" class="mt-2 text-muted small">
            {{ $t("Showing") }} {{ filteredMonitorCount }} {{ $t("of") }} {{ totalMonitorCount }} {{ $t("monitors") }}
        </div>
    </div>

    <!-- Group List -->
    <Draggable
        v-model="filteredGroupList"
        :disabled="!editMode"
        item-key="id"
        :animation="100"
    >
        <template #item="group">
            <div class="mb-5" data-testid="group">
                <!-- Group Title -->
                <h2 class="group-title">
                    <font-awesome-icon v-if="editMode && showGroupDrag" icon="arrows-alt-v" class="action drag me-3" />
                    <font-awesome-icon v-if="editMode" icon="times" class="action remove me-3" @click="removeGroup(group.index)" />
                    <Editable v-model="group.element.name" :contenteditable="editMode" tag="span" data-testid="group-name" />
                </h2>

                <div class="shadow-box monitor-list mt-4 position-relative">
                    <div v-if="group.element.monitorList.length === 0" class="text-center no-monitor-msg">
                        {{ $t("No Monitors") }}
                    </div>

                    <!-- Monitor List -->
                    <!-- animation is not working, no idea why -->
                    <Draggable
                        v-model="group.element.monitorList"
                        class="monitor-list"
                        group="same-group"
                        :disabled="!editMode"
                        :animation="100"
                        item-key="id"
                    >
                        <template #item="monitor">
                            <div class="item" data-testid="monitor">
                                <div class="row">
                                    <div class="col-6 small-padding">
                                        <div class="info">
                                            <font-awesome-icon v-if="editMode" icon="arrows-alt-v" class="action drag me-3" />
                                            <font-awesome-icon v-if="editMode" icon="times" class="action remove me-3" @click="removeMonitor(group.index, monitor.index)" />

                                            <Uptime :monitor="monitor.element" type="24" :pill="true" />
                                            <a
                                                v-if="showLink(monitor)"
                                                :href="monitor.element.url"
                                                class="item-name"
                                                target="_blank"
                                                rel="noopener noreferrer"
                                                data-testid="monitor-name"
                                            >
                                                {{ monitor.element.name }}
                                            </a>
                                            <p v-else class="item-name" data-testid="monitor-name"> {{ monitor.element.name }} </p>

                                            <span
                                                title="Setting"
                                            >
                                                <font-awesome-icon
                                                    v-if="editMode"
                                                    :class="{'link-active': true, 'btn-link': true}"
                                                    icon="cog" class="action me-3"
                                                    data-testid="monitor-settings"
                                                    @click="$refs.monitorSettingDialog.show(group, monitor)"
                                                />
                                            </span>
                                        </div>
                                        <div class="extra-info">
                                            <div v-if="showCertificateExpiry && monitor.element.certExpiryDaysRemaining">
                                                <Tag :item="{name: $t('Cert Exp.'), value: formattedCertExpiryMessage(monitor), color: certExpiryColor(monitor)}" :size="'sm'" />
                                            </div>
                                            <div v-if="showTags">
                                                <Tag v-for="tag in monitor.element.tags" :key="tag" :item="tag" :size="'sm'" data-testid="monitor-tag" />
                                            </div>
                                        </div>
                                    </div>
                                    <div :key="$root.userHeartbeatBar" class="col-6">
                                        <HeartbeatBar size="mid" :monitor-id="monitor.element.id" />
                                    </div>
                                </div>
                            </div>
                        </template>
                    </Draggable>
                </div>
            </div>
        </template>
    </Draggable>
    <MonitorSettingDialog ref="monitorSettingDialog" />
</template>

<script>
import MonitorSettingDialog from "./MonitorSettingDialog.vue";
import Draggable from "vuedraggable";
import HeartbeatBar from "./HeartbeatBar.vue";
import Uptime from "./Uptime.vue";
import Tag from "./Tag.vue";

export default {
    components: {
        MonitorSettingDialog,
        Draggable,
        HeartbeatBar,
        Uptime,
        Tag,
    },
    props: {
        /** Are we in edit mode? */
        editMode: {
            type: Boolean,
            required: true,
        },
        /** Should tags be shown? */
        showTags: {
            type: Boolean,
        },
        /** Should expiry be shown? */
        showCertificateExpiry: {
            type: Boolean,
        }
    },
    data() {
        return {
            searchQuery: "",
            statusFilter: "all",
            originalGroupList: [],
        };
    },
    computed: {
        showGroupDrag() {
            return (this.$root.publicGroupList.length >= 2);
        },
        
        hasActiveFilters() {
            return this.searchQuery.trim() !== "" || this.statusFilter !== "all";
        },
        
        filteredGroupList: {
            get() {
                if (!this.hasActiveFilters) {
                    return this.$root.publicGroupList;
                }
                
                return this.$root.publicGroupList.map(group => {
                    const filteredMonitors = group.monitorList.filter(monitor => {
                        return this.matchesSearch(monitor) && this.matchesStatusFilter(monitor);
                    });
                    
                    return {
                        ...group,
                        monitorList: filteredMonitors
                    };
                }).filter(group => group.monitorList.length > 0);
            },
            set(value) {
                // Handle drag and drop in edit mode
                if (this.editMode) {
                    this.$root.publicGroupList = value;
                }
            }
        },
        
        filteredMonitorCount() {
            return this.filteredGroupList.reduce((count, group) => {
                return count + group.monitorList.length;
            }, 0);
        },
        
        totalMonitorCount() {
            return this.$root.publicGroupList.reduce((count, group) => {
                return count + group.monitorList.length;
            }, 0);
        }
    },
    created() {
        // Store original list for reference
        this.originalGroupList = [...this.$root.publicGroupList];
    },
    methods: {
        /**
         * Check if monitor matches search query
         * @param {object} monitor Monitor to check
         * @returns {boolean} True if monitor matches search
         */
        matchesSearch(monitor) {
            if (!this.searchQuery.trim()) {
                return true;
            }
            
            const query = this.searchQuery.toLowerCase();
            const name = monitor.name.toLowerCase();
            const tags = monitor.tags ? monitor.tags.map(tag => tag.name.toLowerCase()) : [];
            
            return name.includes(query) || tags.some(tag => tag.includes(query));
        },
        
        /**
         * Check if monitor matches status filter
         * @param {object} monitor Monitor to check
         * @returns {boolean} True if monitor matches status filter
         */
        matchesStatusFilter(monitor) {
            if (this.statusFilter === "all") {
                return true;
            }
            
            return monitor.status === this.statusFilter;
        },
        
        /**
         * Filter monitors based on search and status filters
         */
        filterMonitors() {
            // The filtering is handled by the computed property filteredGroupList
            // This method is called on input/change events for reactivity
        },
        
        /**
         * Clear all filters
         */
        clearFilters() {
            this.searchQuery = "";
            this.statusFilter = "all";
        },

        /**
         * Remove the specified group
         * @param {number} index Index of group to remove
         * @returns {void}
         */
        removeGroup(index) {
            this.$root.publicGroupList.splice(index, 1);
        },

        /**
         * Remove a monitor from a group
         * @param {number} groupIndex Index of group to remove monitor
         * from
         * @param {number} index Index of monitor to remove
         * @returns {void}
         */
        removeMonitor(groupIndex, index) {
            this.$root.publicGroupList[groupIndex].monitorList.splice(index, 1);
        },

        /**
         * Should a link to the monitor be shown?
         * Attempts to guess if a link should be shown based upon if
         * sendUrl is set and if the URL is default or not.
         * @param {object} monitor Monitor to check
         * @param {boolean} ignoreSendUrl Should the presence of the sendUrl
         * property be ignored. This will only work in edit mode.
         * @returns {boolean} Should the link be shown
         */
        showLink(monitor, ignoreSendUrl = false) {
            // We must check if there are any elements in monitorList to
            // prevent undefined errors if it hasn't been loaded yet
            if (this.$parent.editMode && ignoreSendUrl && Object.keys(this.$root.monitorList).length) {
                return this.$root.monitorList[monitor.element.id].type === "http" || this.$root.monitorList[monitor.element.id].type === "keyword" || this.$root.monitorList[monitor.element.id].type === "json-query";
            }
            return monitor.element.sendUrl && monitor.element.url && monitor.element.url !== "https://";
        },

        /**
         * Returns formatted certificate expiry or Bad cert message
         * @param {object} monitor Monitor to show expiry for
         * @returns {string} Certificate expiry message
         */
        formattedCertExpiryMessage(monitor) {
            if (monitor?.element?.validCert && monitor?.element?.certExpiryDaysRemaining) {
                return monitor.element.certExpiryDaysRemaining + " " + this.$tc("day", monitor.element.certExpiryDaysRemaining);
            } else if (monitor?.element?.validCert === false) {
                return this.$t("noOrBadCertificate");
            } else {
                return this.$t("Unknown") + " " + this.$tc("day", 2);
            }
        },

        /**
         * Returns certificate expiry color based on days remaining
         * @param {object} monitor Monitor to show expiry for
         * @returns {string} Color for certificate expiry
         */
        certExpiryColor(monitor) {
            if (monitor?.element?.validCert && monitor.element.certExpiryDaysRemaining > 7) {
                return "#059669";
            }
            return "#DC2626";
        },
    }
};
</script>

<style lang="scss" scoped>
@import "../assets/vars";

.search-filter-bar {
    background: #f8f9fa;
    padding: 1rem;
    border-radius: 0.375rem;
    border: 1px solid #dee2e6;
}

.extra-info {
    display: flex;
    margin-bottom: 0.5rem;
}

.extra-info > div > div:first-child {
    margin-left: 0 !important;
}

.no-monitor-msg {
    position: absolute;
    width: 100%;
    top: 20px;
    left: 0;
}

.monitor-list {
    min-height: 46px;
}

.item-name {
    padding-left: 5px;
    padding-right: 5px;
    margin: 0;
    display: inline-block;
}

.btn-link {
    color: #bbbbbb;
    margin-left: 5px;
}

.link-active {
    color: $primary;
}

.flip-list-move {
    transition: transform 0.5s;
}

.no-move {
    transition: transform 0s;
}

.drag {
    color: #bbb;
    cursor: grab;
}

.remove {
    color: $danger;
}

.group-title {
    span {
        display: inline-block;
        min-width: 15px;
    }
}

.mobile {
    .item {
        padding: 13px 0 10px;
    }
}

.bg-maintenance {
    background-color: $maintenance;
}

</style>
