<template>
    <!-- Search and Filter Bar -->
    <div v-if="!editMode" class="search-filter-bar mb-4">
        <div class="row g-3">
            <!-- Search Input -->
            <div class="col-12 col-md-6">
                <div class="input-group search-input-group">
                    <span class="input-group-text search-icon">
                        <font-awesome-icon icon="search" />
                    </span>
                    <input 
                        v-model="searchQuery" 
                        type="text" 
                        class="form-control search-input" 
                        :placeholder="$t('Pesquisar')"
                        @input="filterMonitors"
                    />
                </div>
            </div>
            
            <!-- Status Filter -->
            <div class="col-12 col-md-3">
                                 <select v-model="statusFilter" class="form-select status-filter" @change="filterMonitors">
                     <option value="all">{{ $t("All Status") }}</option>
                     <option value="1">{{ $t("Up") }}</option>
                     <option value="0">{{ $t("Down") }}</option>
                     <option value="3">{{ $t("Maintenance") }}</option>
                 </select>
            </div>
            
            <!-- Clear Filters Button -->
            <div class="col-12 col-md-3">
                <button 
                    v-if="hasActiveFilters" 
                    class="btn btn-outline-light clear-filters-btn w-100" 
                    @click="clearFilters"
                >
                    <font-awesome-icon icon="times" class="me-2" />
                    {{ $t("Clear Filters") }}
                </button>
            </div>
        </div>
        
        <!-- Results counter -->
        <div v-if="hasActiveFilters" class="mt-3 text-light results-counter">
            <small>
                {{ $t("Showing") }} <strong>{{ filteredMonitorCount }}</strong> {{ $t("of") }} <strong>{{ totalMonitorCount }}</strong> {{ $t("monitors") }}
            </small>
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
                                    <div class="col-12 col-md-6 small-padding">
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
                                    <div :key="$root.userHeartbeatBar" class="col-12 col-md-6">
                                        <HeartbeatBar size="mid" :monitor-id="monitor.element.id" :show-time-on-mobile="true" />
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
             
             // Get the actual status from lastHeartbeatList
             let actualStatus = monitor.status;
             if (monitor.id in this.$root.publicLastHeartbeatList && this.$root.publicLastHeartbeatList[monitor.id]) {
                 actualStatus = this.$root.publicLastHeartbeatList[monitor.id].status;
             }
             
             // Convert statusFilter to number for comparison
             const filterStatus = parseInt(this.statusFilter);
             
             return actualStatus === filterStatus;
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

/* Permitir quebra de linha no container de info da página pública */
.info {
    white-space: normal !important;
    overflow: visible;
}

.search-filter-bar {
    background: transparent;
    padding: 1rem 0;
    border-radius: 0.375rem;
}

.search-input-group {
    .search-icon {
        background: var(--search-bg, rgba(255, 255, 255, 0.1));
        border: 1px solid var(--search-border, rgba(255, 255, 255, 0.2));
        border-right: none;
        color: var(--search-icon-color, rgba(255, 255, 255, 0.8));
    }
    
    .search-input {
        background: var(--search-bg, rgba(255, 255, 255, 0.1));
        border: 1px solid var(--search-border, rgba(255, 255, 255, 0.2));
        color: var(--search-text-color, white);
        
        &::placeholder {
            color: var(--search-placeholder, rgba(255, 255, 255, 0.6));
        }
        
        &:focus {
            background: var(--search-focus-bg, rgba(255, 255, 255, 0.15));
            border-color: var(--search-focus-border, rgba(255, 255, 255, 0.3));
            box-shadow: 0 0 0 0.2rem var(--search-focus-shadow, rgba(255, 255, 255, 0.1));
            color: var(--search-text-color, white);
        }
    }
}

.status-filter {
    background: var(--search-bg, rgba(255, 255, 255, 0.1));
    border: 1px solid var(--search-border, rgba(255, 255, 255, 0.2));
    color: var(--search-text-color, white);
    
    &:focus {
        background: var(--search-focus-bg, rgba(255, 255, 255, 0.15));
        border-color: var(--search-focus-border, rgba(255, 255, 255, 0.3));
        box-shadow: 0 0 0 0.2rem var(--search-focus-shadow, rgba(255, 255, 255, 0.1));
        color: var(--search-text-color, white);
    }
    
    option {
        background: var(--option-bg, #2d3748);
        color: var(--option-text, white);
    }
}

.clear-filters-btn {
    border: 1px solid var(--btn-border, rgba(255, 255, 255, 0.3));
    color: var(--btn-text, rgba(255, 255, 255, 0.8));
    background: var(--btn-bg, rgba(255, 255, 255, 0.1));
    
    &:hover {
        background: var(--btn-hover-bg, rgba(255, 255, 255, 0.2));
        border-color: var(--btn-hover-border, rgba(255, 255, 255, 0.4));
        color: var(--btn-hover-text, white);
    }
}

.results-counter {
    opacity: 0.8;
    color: var(--counter-text, white);
    
    strong {
        color: var(--counter-strong, #4ade80);
    }
}

/* Dark theme (default) */
.search-filter-bar {
    --search-bg: rgba(255, 255, 255, 0.1);
    --search-border: rgba(255, 255, 255, 0.2);
    --search-icon-color: rgba(255, 255, 255, 0.8);
    --search-text-color: white;
    --search-placeholder: rgba(255, 255, 255, 0.6);
    --search-focus-bg: rgba(255, 255, 255, 0.15);
    --search-focus-border: rgba(255, 255, 255, 0.3);
    --search-focus-shadow: rgba(255, 255, 255, 0.1);
    --option-bg: #2d3748;
    --option-text: white;
    --btn-border: rgba(255, 255, 255, 0.3);
    --btn-text: rgba(255, 255, 255, 0.8);
    --btn-bg: rgba(255, 255, 255, 0.1);
    --btn-hover-bg: rgba(255, 255, 255, 0.2);
    --btn-hover-border: rgba(255, 255, 255, 0.4);
    --btn-hover-text: white;
    --counter-text: white;
    --counter-strong: #4ade80;
}

/* Light theme - detect system preference */
@media (prefers-color-scheme: light) {
    .search-filter-bar {
        --search-bg: rgba(0, 0, 0, 0.08);
        --search-border: rgba(0, 0, 0, 0.15);
        --search-icon-color: rgba(0, 0, 0, 0.7);
        --search-text-color: #000;
        --search-placeholder: rgba(0, 0, 0, 0.5);
        --search-focus-bg: rgba(0, 0, 0, 0.12);
        --search-focus-border: rgba(0, 0, 0, 0.25);
        --search-focus-shadow: rgba(0, 0, 0, 0.15);
        --option-bg: #ffffff;
        --option-text: #000;
        --btn-border: rgba(0, 0, 0, 0.15);
        --btn-text: rgba(0, 0, 0, 0.8);
        --btn-bg: rgba(0, 0, 0, 0.08);
        --btn-hover-bg: rgba(0, 0, 0, 0.15);
        --btn-hover-border: rgba(0, 0, 0, 0.25);
        --btn-hover-text: #000;
        --counter-text: #333;
        --counter-strong: #059669;
    }
}

/* Force light theme when Uptime Kuma is in light mode */
body[data-theme="light"] .search-filter-bar {
    --search-bg: rgba(0, 0, 0, 0.08);
    --search-border: rgba(0, 0, 0, 0.15);
    --search-icon-color: rgba(0, 0, 0, 0.7);
    --search-text-color: #000;
    --search-placeholder: rgba(0, 0, 0, 0.5);
    --search-focus-bg: rgba(0, 0, 0, 0.12);
    --search-focus-border: rgba(0, 0, 0, 0.25);
    --search-focus-shadow: rgba(0, 0, 0, 0.15);
    --option-bg: #ffffff;
    --option-text: #000;
    --btn-border: rgba(0, 0, 0, 0.15);
    --btn-text: rgba(0, 0, 0, 0.8);
    --btn-bg: rgba(0, 0, 0, 0.08);
    --btn-hover-bg: rgba(0, 0, 0, 0.15);
    --btn-hover-border: rgba(0, 0, 0, 0.25);
    --btn-hover-text: #000;
    --counter-text: #333;
    --counter-strong: #059669;
}

/* Force dark theme when Uptime Kuma is in dark mode */
body[data-theme="dark"] .search-filter-bar {
    --search-bg: rgba(255, 255, 255, 0.1);
    --search-border: rgba(255, 255, 255, 0.2);
    --search-icon-color: rgba(255, 255, 255, 0.8);
    --search-text-color: white;
    --search-placeholder: rgba(255, 255, 255, 0.6);
    --search-focus-bg: rgba(255, 255, 255, 0.15);
    --search-focus-border: rgba(255, 255, 255, 0.3);
    --search-focus-shadow: rgba(255, 255, 255, 0.1);
    --option-bg: #2d3748;
    --option-text: white;
    --btn-border: rgba(255, 255, 255, 0.3);
    --btn-text: rgba(255, 255, 255, 0.8);
    --btn-bg: rgba(255, 255, 255, 0.1);
    --btn-hover-bg: rgba(255, 255, 255, 0.2);
    --btn-hover-border: rgba(255, 255, 255, 0.4);
    --btn-hover-text: white;
    --counter-text: white;
    --counter-strong: #4ade80;
}

/* Fallback styles for better visibility */
.search-filter-bar .search-input-group .search-icon {
    background: #f8f9fa !important;
    border: 1px solid #dee2e6 !important;
    color: #6c757d !important;
}

.search-filter-bar .search-input-group .search-input {
    background: #ffffff !important;
    border: 1px solid #dee2e6 !important;
    color: #212529 !important;
}

.search-filter-bar .search-input-group .search-input::placeholder {
    color: #6c757d !important;
}

.search-filter-bar .status-filter {
    background: #ffffff !important;
    border: 1px solid #dee2e6 !important;
    color: #212529 !important;
}

.search-filter-bar .status-filter option {
    background: #ffffff !important;
    color: #212529 !important;
}

.search-filter-bar .clear-filters-btn {
    background: #6c757d !important;
    border: 1px solid #6c757d !important;
    color: #ffffff !important;
}

.search-filter-bar .clear-filters-btn:hover {
    background: #5a6268 !important;
    border-color: #545b62 !important;
    color: #ffffff !important;
}

.search-filter-bar .results-counter {
    color: #6c757d !important;
}

.search-filter-bar .results-counter strong {
    color: #28a745 !important;
}

/* Dark theme overrides */
body[data-theme="dark"] .search-filter-bar .search-input-group .search-icon,
body[data-theme="dark"] .search-filter-bar .search-input-group .search-input,
body[data-theme="dark"] .search-filter-bar .status-filter {
    background: rgba(255, 255, 255, 0.1) !important;
    border: 1px solid rgba(255, 255, 255, 0.2) !important;
    color: white !important;
}

body[data-theme="dark"] .search-filter-bar .search-input-group .search-input::placeholder {
    color: rgba(255, 255, 255, 0.6) !important;
}

body[data-theme="dark"] .search-filter-bar .status-filter option {
    background: #2d3748 !important;
    color: white !important;
}

body[data-theme="dark"] .search-filter-bar .clear-filters-btn {
    background: rgba(255, 255, 255, 0.1) !important;
    border: 1px solid rgba(255, 255, 255, 0.3) !important;
    color: rgba(255, 255, 255, 0.8) !important;
}

body[data-theme="dark"] .search-filter-bar .clear-filters-btn:hover {
    background: rgba(255, 255, 255, 0.2) !important;
    border-color: rgba(255, 255, 255, 0.4) !important;
    color: white !important;
}

body[data-theme="dark"] .search-filter-bar .results-counter {
    color: white !important;
}

body[data-theme="dark"] .search-filter-bar .results-counter strong {
    color: #4ade80 !important;
}

/* Mobile Responsive */
@media (max-width: 768px) {
    .search-filter-bar {
        padding: 0.75rem 0;
    }
    
    .search-input-group,
    .status-filter,
    .clear-filters-btn {
        margin-bottom: 0.5rem;
    }
    
    .results-counter {
        text-align: center;
    }
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
    white-space: normal; /* permitir quebra de linha no público */
    word-break: break-word;
}

/* Ajustes de responsividade: forçar stack no mobile */
@media (max-width: 768px) {
    .item .row {
        display: flex;
        flex-direction: column;
    }
    .item .col-6.small-padding {
        width: 100% !important;
    }
    .item .col-6 {
        width: 100% !important;
        margin-top: 8px;
    }
    .info {
        white-space: normal !important; /* sobrescreve nowrap global apenas aqui */
    }
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
