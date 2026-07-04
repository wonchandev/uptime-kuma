<template>
    <!-- Group List -->
    <Draggable v-model="$root.publicGroupList" :disabled="!editMode" item-key="id" :animation="100">
        <template #item="group">
            <div class="mb-4" data-testid="group">
                <!-- Group Title -->
                <h2 class="group-title">
                    <font-awesome-icon v-if="editMode && showGroupDrag" icon="arrows-alt-v" class="action drag me-3" />
                    <font-awesome-icon
                        v-if="editMode"
                        icon="times"
                        class="action remove me-3"
                        @click="removeGroup(group.index)"
                    />
                    <Editable
                        v-model="group.element.name"
                        :contenteditable="editMode"
                        tag="span"
                        data-testid="group-name"
                    />
                </h2>

                <div class="shadow-box monitor-list mt-3 position-relative">
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
                                        <div class="col-9 col-xl-6 small-padding">
                                            <div class="info">
                                                <font-awesome-icon
                                                    v-if="editMode"
                                                    icon="arrows-alt-v"
                                                    class="action drag me-3"
                                                />
                                                <font-awesome-icon
                                                    v-if="editMode"
                                                    icon="times"
                                                    class="action remove me-3"
                                                    @click="removeMonitor(group.index, monitor.index)"
                                                />

                                                <font-awesome-icon
                                                    v-if="editMode"
                                                    icon="cog"
                                                    class="action me-3 ms-0"
                                                    :class="{ 'link-active': true, 'btn-link': true }"
                                                    data-testid="monitor-settings"
                                                    @click="$refs.monitorSettingDialog.show(group, monitor)"
                                                />
                                                <Status
                                                    v-if="showOnlyLastHeartbeat"
                                                    :status="statusOfLastHeartbeat(monitor.element.id)"
                                                />
                                                <Uptime v-else :monitor="monitor.element" type="24" :pill="true" />
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
                                                <p v-else class="item-name" data-testid="monitor-name">
                                                    {{ monitor.element.name }}
                                                </p>
                                                <span
                                                    v-if="lastPing(monitor.element.id) !== null"
                                                    class="ping-badge"
                                                >
                                                    {{ lastPing(monitor.element.id) }}ms
                                                </span>
                                            </div>
                                            <div class="extra-info">
                                                <div
                                                    v-if="
                                                        showCertificateExpiry && monitor.element.certExpiryDaysRemaining
                                                    "
                                                >
                                                    <Tag
                                                        :item="{
                                                            name: $t('Cert Exp.'),
                                                            value: formattedCertExpiryMessage(monitor),
                                                            color: certExpiryColor(monitor),
                                                        }"
                                                        :size="'sm'"
                                                    />
                                                </div>
                                                <div v-if="showTags">
                                                    <Tag
                                                        v-for="tag in monitor.element.tags"
                                                        :key="tag"
                                                        :item="tag"
                                                        :size="'sm'"
                                                        data-testid="monitor-tag"
                                                    />
                                                </div>
                                            </div>
                                        </div>
                                        <div :key="$root.userHeartbeatBar" class="col-3 col-xl-6 beat-col">
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
import Status from "./Status.vue";

export default {
    components: {
        MonitorSettingDialog,
        Draggable,
        HeartbeatBar,
        Uptime,
        Tag,
        Status,
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
        },
        /** Should only the last heartbeat be shown? */
        showOnlyLastHeartbeat: {
            type: Boolean,
        },
    },
    data() {
        return {};
    },
    computed: {
        showGroupDrag() {
            return this.$root.publicGroupList.length >= 2;
        },
    },
    methods: {
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
         * @param {number} groupIndex Index of group to remove monitor from
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
                return (
                    this.$root.monitorList[monitor.element.id].type === "http" ||
                    this.$root.monitorList[monitor.element.id].type === "keyword" ||
                    this.$root.monitorList[monitor.element.id].type === "json-query"
                );
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
                return this.$t("days", monitor.element.certExpiryDaysRemaining);
            } else if (monitor?.element?.validCert === false) {
                return this.$t("noOrBadCertificate");
            } else {
                return this.$t("unknownDays");
            }
        },

        /**
         * Returns the most recent response time of a monitor
         * @param {number} monitorId Id of the monitor
         * @returns {number|null} Latest ping in ms, or null if unknown
         */
        lastPing(monitorId) {
            const heartbeats = this.$root.heartbeatList[monitorId] ?? [];
            const lastBeat = heartbeats[heartbeats.length - 1];
            return typeof lastBeat?.ping === "number" ? lastBeat.ping : null;
        },

        /**
         * Returns the status of the last heartbeat
         * @param {number} monitorId Id of the monitor to get status for
         * @returns {number} Status of the last heartbeat
         */
        statusOfLastHeartbeat(monitorId) {
            let heartbeats = this.$root.heartbeatList[monitorId] ?? [];
            let lastHeartbeat = heartbeats[heartbeats.length - 1];
            return lastHeartbeat?.status;
        },

        /**
         * Returns certificate expiry color based on days remaining
         * @param {object} monitor Monitor to show expiry for
         * @returns {string} Color for certificate expiry
         */
        certExpiryColor(monitor) {
            if (monitor?.element?.validCert && monitor.element.certExpiryDaysRemaining > 7) {
                return "#017531";
            }
            return "#DC2626";
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars";

.extra-info {
    display: flex;
    margin-top: 0.25rem;

    // Hide when there are no tags so the name column stays perfectly centered
    &:empty {
        display: none;
    }
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

// Flatten the old boxed group container — cards carry the visual weight now
.shadow-box.monitor-list {
    background-color: transparent !important;
    box-shadow: none;
    padding: 0;
    border-radius: 0;
}

// Each monitor becomes a slim card separated by whitespace.
// --heartbeat-up-color is picked up by the HeartbeatBar canvas (Toss blue).
.monitor-list .item {
    --heartbeat-up-color: #017531;

    background-color: #fff;
    border-radius: 16px;
    padding: 10px 16px;
    margin-bottom: 12px;
    box-shadow: 0 1px 2px rgba(23, 32, 42, 0.03), 0 4px 16px rgba(23, 32, 42, 0.04);

    // Vertically center the name column against the taller heartbeat column
    .row {
        align-items: center;
    }

    // No hover effect — keep the same look as the resting state
    // (also neutralizes the global .monitor-list .item:hover highlight)
    &:hover {
        background-color: #fff;
    }

    .dark & {
        --heartbeat-up-color: #12a35b;

        background-color: #1e1e24;
        box-shadow: none;

        &:hover {
            background-color: #1e1e24;
        }
    }
}

.item-name {
    padding-left: 5px;
    padding-right: 5px;
    margin: 0;
    display: inline-block;
    font-weight: 600;
    color: #191f28;

    .dark & {
        color: #fff;
    }
}

a.item-name:hover {
    text-decoration: underline;
}

// Real-time response time capsule next to the service name
.ping-badge {
    display: inline-block;
    vertical-align: middle;
    margin-left: 6px;
    font-size: 11px;
    font-weight: 700;
    font-variant-numeric: tabular-nums;
    line-height: 1;
    padding: 4px 8px;
    border-radius: 50rem;
    background-color: rgba(1, 117, 49, 0.09);
    color: #017531;

    .dark & {
        background-color: rgba(18, 163, 91, 0.16);
        color: #34c17e;
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
    font-size: 17px;
    font-weight: 700;
    letter-spacing: -0.2px;
    color: #4e5968;
    padding-left: 4px;

    .dark & {
        color: #c3c9d1;
    }

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

<!-- eslint-disable-next-line vue-scoped-css/enforce-style-type -->
<style lang="scss">
@import "../assets/vars";

// Uptime pill → soft tinted capsule with vivid text.
// Unscoped because the badge is rendered by the child Uptime component;
// limited to status pages via the body class set by StatusPage.
body.status-page-body .monitor-list .item .badge.rounded-pill {
    font-weight: 700;
    font-variant-numeric: tabular-nums;

    &.bg-primary {
        background-color: rgba(1, 117, 49, 0.09) !important;
        color: #017531;
    }

    &.bg-danger {
        background-color: rgba(220, 53, 69, 0.1) !important;
        color: #dc3545;
    }

    &.bg-warning {
        background-color: rgba(248, 163, 6, 0.12) !important;
        color: #b45309;
    }

    &.bg-maintenance {
        background-color: rgba(23, 71, 245, 0.1) !important;
        color: $maintenance;
    }
}

body.status-page-body.dark .monitor-list .item .badge.rounded-pill {
    &.bg-primary {
        background-color: rgba(18, 163, 91, 0.16) !important;
        color: #34c17e;
    }

    &.bg-danger {
        background-color: rgba(255, 107, 129, 0.14) !important;
        color: #ff6b81;
    }

    &.bg-warning {
        background-color: rgba(248, 163, 6, 0.14) !important;
        color: #fbbf24;
    }

    &.bg-maintenance {
        background-color: rgba(61, 139, 253, 0.14) !important;
        color: #7db4ff;
    }
}
</style>
