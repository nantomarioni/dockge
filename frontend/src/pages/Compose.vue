<template>
    <transition name="slide-fade" appear>
        <div>
            <h1 v-if="isAdd" class="mb-3">{{ $t("compose") }}</h1>
            <h1 v-else class="mb-3">
                <Uptime :stack="globalStack" :pill="true" /> {{ stack.name }}
                <span v-if="$root.agentCount > 1" class="agent-name">
                    ({{ endpointDisplay }})
                </span>
            </h1>

            <div v-if="stack.isManagedByDockge" class="mb-3">
                <div class="btn-group me-2" role="group">
                    <button v-if="isEditMode" class="btn btn-primary" :disabled="processing" @click="deployStack">
                        <font-awesome-icon icon="rocket" class="me-1" />
                        {{ $t("deployStack") }}
                    </button>

                    <button v-if="isEditMode" class="btn btn-normal" :disabled="processing" @click="saveStack">
                        <font-awesome-icon icon="save" class="me-1" />
                        {{ $t("saveStackDraft") }}
                    </button>

                    <button v-if="!isEditMode" class="btn btn-secondary" :disabled="processing" @click="enableEditMode">
                        <font-awesome-icon icon="pen" class="me-1" />
                        {{ $t("editStack") }}
                    </button>

                    <button v-if="!isEditMode && !active" class="btn btn-primary" :disabled="processing" @click="startStack">
                        <font-awesome-icon icon="play" class="me-1" />
                        {{ $t("startStack") }}
                    </button>

                    <button v-if="!isEditMode && active" class="btn btn-normal " :disabled="processing" @click="restartStack">
                        <font-awesome-icon icon="rotate" class="me-1" />
                        {{ $t("restartStack") }}
                    </button>

                    <button v-if="!isEditMode" class="btn btn-normal" :disabled="processing" @click="updateStack">
                        <font-awesome-icon icon="cloud-arrow-down" class="me-1" />
                        {{ $t("updateStack") }}
                    </button>

                    <button v-if="!isEditMode && active" class="btn btn-normal" :disabled="processing" @click="stopStack">
                        <font-awesome-icon icon="stop" class="me-1" />
                        {{ $t("stopStack") }}
                    </button>

                    <BDropdown right text="" variant="normal">
                        <BDropdownItem @click="downStack">
                            <font-awesome-icon icon="stop" class="me-1" />
                            {{ $t("downStack") }}
                        </BDropdownItem>
                    </BDropdown>
                </div>

                <button v-if="isEditMode && !isAdd" class="btn btn-normal" :disabled="processing" @click="discardStack">{{ $t("discardStack") }}</button>
                <button v-if="!isEditMode" class="btn btn-danger" :disabled="processing" @click="showDeleteDialog = !showDeleteDialog">
                    <font-awesome-icon icon="trash" class="me-1" />
                    {{ $t("deleteStack") }}
                </button>
            </div>

            <!-- URLs -->
            <div v-if="urls.length > 0" class="mb-3">
                <a v-for="(url, index) in urls" :key="index" target="_blank" :href="url.url">
                    <span class="badge bg-secondary me-2">{{ url.display }}</span>
                </a>
            </div>

            <!-- Progress Terminal -->
            <transition name="slide-fade" appear>
                <Terminal
                    v-show="showProgressTerminal"
                    ref="progressTerminal"
                    class="mb-3 terminal"
                    :name="terminalName"
                    :endpoint="endpoint"
                    :rows="progressTerminalRows"
                    @has-data="showProgressTerminal = true; submitted = true;"
                ></Terminal>
            </transition>

            <div v-if="stack.isManagedByDockge" class="row">
                <div class="col-lg-12">
                    <!-- General -->
                    <div v-if="isAdd">
                        <h4 class="mb-3">{{ $t("general") }}</h4>
                        <div class="shadow-box big-padding mb-3">
                            <!-- Stack Name -->
                            <div>
                                <label for="name" class="form-label">{{ $t("stackName") }}</label>
                                <input id="name" v-model="stack.name" type="text" class="form-control" required @blur="stackNameToLowercase">
                                <div class="form-text">{{ $t("Lowercase only") }}</div>
                            </div>

                            <!-- Endpoint -->
                            <div class="mt-3">
                                <label for="name" class="form-label">{{ $t("dockgeAgent") }}</label>
                                <select v-model="stack.endpoint" class="form-select">
                                    <option v-for="(agent, endpoint) in $root.agentList" :key="endpoint" :value="endpoint" :disabled="$root.agentStatusList[endpoint] != 'online'">
                                        ({{ $root.agentStatusList[endpoint] }}) {{ (endpoint) ? endpoint : $t("currentEndpoint") }}
                                    </option>
                                </select>
                            </div>
                        </div>
                    </div>

                    <!-- Combined Terminal Output -->
                    <div v-show="!isEditMode">
                        <div class="d-flex justify-content-between align-items-center mb-3">
                            <h4 class="mb-0">{{ $t("terminal") }}</h4>
                            <div class="form-check form-switch">
                                <input 
                                    id="timestampToggle" 
                                    v-model="showTimestamps" 
                                    class="form-check-input" 
                                    type="checkbox"
                                    @change="onTimestampToggle"
                                >
                                <label class="form-check-label" for="timestampToggle">
                                    Show Timestamps
                                </label>
                            </div>
                        </div>
                        <Terminal
                            ref="combinedTerminal"
                            class="mb-3 terminal"
                            :name="combinedTerminalName"
                            :endpoint="endpoint"
                            :rows="combinedTerminalRows"
                            :cols="combinedTerminalCols"
                            style="height: 315px;"
                        ></Terminal>
                    </div>

                    <h4 class="mb-3">{{ stack.composeFileName }}</h4>

                    <!-- YAML editor -->
                    <div class="shadow-box mb-3 editor-box" :class="{'edit-mode' : isEditMode}">
                        <prism-editor
                            ref="editor"
                            v-model="stack.composeYAML"
                            class="yaml-editor"
                            :highlight="highlighterYAML"
                            line-numbers :readonly="!isEditMode"
                            @focus="editorFocus = true"
                            @blur="editorFocus = false"
                        ></prism-editor>
                    </div>
                    <div v-if="isEditMode" class="mb-3">
                        {{ yamlError }}
                    </div>

                    <!-- ENV editor -->
                    <div v-if="isEditMode">
                        <h4 class="mb-3">.env</h4>
                        <div class="shadow-box mb-3 editor-box" :class="{'edit-mode' : isEditMode}">
                            <prism-editor
                                ref="editor"
                                v-model="stack.composeENV"
                                class="env-editor"
                                :highlight="highlighterENV"
                                line-numbers :readonly="!isEditMode"
                                @focus="editorFocus = true"
                                @blur="editorFocus = false"
                            ></prism-editor>
                        </div>
                    </div>

                    <div v-if="isEditMode">
                        <div class="shadow-box big-padding mb-3">
                            <!-- URLs -->
                            <div class="mb-4">
                                <label class="form-label">
                                    {{ $tc("url", 2) }}
                                </label>
                                <ArrayInput name="urls" :display-name="$t('url')" placeholder="https://" object-type="x-dockge" />
                            </div>
                        </div>
                    </div>
                </div>
            </div>

            <div v-if="!stack.isManagedByDockge && !processing">
                {{ $t("stackNotManagedByDockgeMsg") }}
            </div>

            <!-- Delete Dialog -->
            <BModal v-model="showDeleteDialog" :cancelTitle="$t('cancel')" :okTitle="$t('deleteStack')" okVariant="danger" @ok="deleteDialog">
                {{ $t("deleteStackMsg") }}
            </BModal>
        </div>
    </transition>
</template>

<script>
import { highlight, languages } from "prismjs/components/prism-core";
import { PrismEditor } from "vue-prism-editor";
import "prismjs/components/prism-yaml";

import "prismjs/themes/prism-tomorrow.css";
import "vue-prism-editor/dist/prismeditor.min.css";
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome";
import {
    COMBINED_TERMINAL_COLS,
    COMBINED_TERMINAL_ROWS,
    copyYAMLComments, envsubstYAML,
    getCombinedTerminalName,
    getComposeTerminalName,
    PROGRESS_TERMINAL_ROWS,
    RUNNING
} from "../../../common/util-common";
import { BModal } from "bootstrap-vue-next";
import dotenv from "dotenv";

const template = `
services:
  nginx:
    image: nginx:latest
    restart: unless-stopped
    ports:
      - "8080:80"
`;
const envDefault = "# VARIABLE=value #comment";

let yamlErrorTimeout = null;

let serviceStatusTimeout = null;
let prismjsSymbolDefinition = {
    "symbol": {
        pattern: /(?<!\$)\$(\{[^{}]*\}|\w+)/,
    }
};

export default {
    components: {
        FontAwesomeIcon,
        PrismEditor,
        BModal,
    },
    beforeRouteUpdate(to, from, next) {
        this.exitConfirm(next);
    },
    beforeRouteLeave(to, from, next) {
        this.exitConfirm(next);
    },

    data() {
        return {
            editorFocus: false,
            envsubstJSONConfig: {},
            yamlError: "",
            processing: true,
            showProgressTerminal: false,
            progressTerminalRows: PROGRESS_TERMINAL_ROWS,
            combinedTerminalRows: COMBINED_TERMINAL_ROWS,
            combinedTerminalCols: COMBINED_TERMINAL_COLS,
            stack: {

            },
            serviceStatusList: {},
            isEditMode: false,
            submitted: false,
            showDeleteDialog: false,

            stopServiceStatusTimeout: false,
            showTimestamps: localStorage.getItem("showTimestamps") === "true",
        };
    },
    computed: {

        endpointDisplay() {
            return this.$root.endpointDisplayFunction(this.endpoint);
        },

        urls() {
            if (!this.envsubstJSONConfig["x-dockge"] || !this.envsubstJSONConfig["x-dockge"].urls || !Array.isArray(this.envsubstJSONConfig["x-dockge"].urls)) {
                return [];
            }

            let urls = [];
            for (const url of this.envsubstJSONConfig["x-dockge"].urls) {
                let display;
                try {
                    let obj = new URL(url);
                    let pathname = obj.pathname;
                    if (pathname === "/") {
                        pathname = "";
                    }
                    display = obj.host + pathname + obj.search;
                } catch (e) {
                    display = url;
                }

                urls.push({
                    display,
                    url,
                });
            }
            return urls;
        },

        isAdd() {
            return this.$route.path === "/compose" && !this.submitted;
        },

        /**
         * Get the stack from the global stack list, because it may contain more real-time data like status
         * @return {*}
         */
        globalStack() {
            return this.$root.completeStackList[this.stack.name + "_" + this.endpoint];
        },

        status() {
            return this.globalStack?.status;
        },

        active() {
            return this.status === RUNNING;
        },

        terminalName() {
            if (!this.stack.name) {
                return "";
            }
            return getComposeTerminalName(this.endpoint, this.stack.name);
        },

        combinedTerminalName() {
            if (!this.stack.name) {
                return "";
            }
            return getCombinedTerminalName(this.endpoint, this.stack.name);
        },



        endpoint() {
            return this.stack.endpoint || this.$route.params.endpoint || "";
        },

        url() {
            if (this.stack.endpoint) {
                return `/compose/${this.stack.name}/${this.stack.endpoint}`;
            } else {
                return `/compose/${this.stack.name}`;
            }
        },
    },
    watch: {
        "stack.composeYAML": {
            handler() {
                // Removed smart UI logic
            },
            deep: true,
        },

        "stack.composeENV": {
            handler() {
               // Removed smart UI logic
            },
            deep: true,
        },

        $route(to, from) {

        }
    },
    mounted() {
        if (this.isAdd) {
            this.processing = false;
            this.isEditMode = true;

            let composeYAML;
            let composeENV;

            if (this.$root.composeTemplate) {
                composeYAML = this.$root.composeTemplate;
                this.$root.composeTemplate = "";
            } else {
                composeYAML = template;
            }
            if (this.$root.envTemplate) {
                composeENV = this.$root.envTemplate;
                this.$root.envTemplate = "";
            } else {
                composeENV = envDefault;
            }

            // Default Values
            this.stack = {
                name: "",
                composeYAML,
                composeENV,
                isManagedByDockge: true,
                endpoint: "",
            };

            this.stack = {
                name: "",
                composeYAML,
                composeENV,
                isManagedByDockge: true,
                endpoint: "",
            };

        } else {
            this.stack.name = this.$route.params.stackName;
            this.loadStack();
        }

        this.requestServiceStatus();
    },
    unmounted() {

    },
    methods: {
        startServiceStatusTimeout() {
            clearTimeout(serviceStatusTimeout);
            serviceStatusTimeout = setTimeout(async () => {
                this.requestServiceStatus();
            }, 5000);
        },

        requestServiceStatus() {
            // Do not request if it is add mode
            if (this.isAdd) {
                return;
            }

            this.$root.emitAgent(this.endpoint, "serviceStatusList", this.stack.name, (res) => {
                if (res.ok) {
                    this.serviceStatusList = res.serviceStatusList;
                }
                if (!this.stopServiceStatusTimeout) {
                    this.startServiceStatusTimeout();
                }
            });
        },

        exitConfirm(next) {
            if (this.isEditMode) {
                if (confirm(this.$t("confirmLeaveStack"))) {
                    this.exitAction();
                    next();
                } else {
                    next(false);
                }
            } else {
                this.exitAction();
                next();
            }
        },

        exitAction() {
            console.log("exitAction");
            this.stopServiceStatusTimeout = true;
            clearTimeout(serviceStatusTimeout);

            // Leave Combined Terminal
            console.debug("leaveCombinedTerminal", this.endpoint, this.stack.name);
            this.$root.emitAgent(this.endpoint, "leaveCombinedTerminal", this.stack.name, () => {});
        },

        bindTerminal() {
            this.$refs.progressTerminal?.bind(this.endpoint, this.terminalName);
        },

        loadStack() {
            this.processing = true;
            this.$root.emitAgent(this.endpoint, "getStack", this.stack.name, (res) => {
                if (res.ok) {

                    this.stack = res.stack;
                    this.processing = false;
                    this.bindTerminal();
                    
                    // Join combined terminal with timestamp preference
                    this.$root.emitAgent(this.endpoint, "joinCombinedTerminal", this.stack.name, this.showTimestamps, (termRes) => {
                        if (!termRes.ok) {
                            this.$root.toastRes(termRes);
                        }
                    });
                } else {
                    this.$root.toastRes(res);
                }
            });
        },

        deployStack() {
            this.processing = true;

            // Check if services is object
            // Removed jsonConfig check

            // Set the stack name if empty, use the first container name
            if (!this.stack.name) {
                 this.$root.toastError("Stack name cannot be empty");
                 this.processing = false;
                 return;
            }

            this.bindTerminal();

            this.$root.emitAgent(this.stack.endpoint, "deployStack", this.stack.name, this.stack.composeYAML, this.stack.composeENV, this.isAdd, (res) => {
                this.processing = false;
                this.$root.toastRes(res);

                if (res.ok) {
                    this.isEditMode = false;
                    this.$router.push(this.url);
                    
                    // Join combined terminal with timestamp preference
                    this.$root.emitAgent(this.endpoint, "joinCombinedTerminal", this.stack.name, this.showTimestamps, (termRes) => {
                        if (!termRes.ok) {
                            this.$root.toastRes(termRes);
                        }
                    });
                }
            });
        },

        saveStack() {
            this.processing = true;

            this.$root.emitAgent(this.stack.endpoint, "saveStack", this.stack.name, this.stack.composeYAML, this.stack.composeENV, this.isAdd, (res) => {
                this.processing = false;
                this.$root.toastRes(res);

                if (res.ok) {
                    this.isEditMode = false;
                    this.$router.push(this.url);
                }
            });
        },

        startStack() {
            this.processing = true;

            this.$root.emitAgent(this.endpoint, "startStack", this.stack.name, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                
                if (res.ok) {
                    // Join combined terminal with timestamp preference
                    this.$root.emitAgent(this.endpoint, "joinCombinedTerminal", this.stack.name, this.showTimestamps, (termRes) => {
                        if (!termRes.ok) {
                            this.$root.toastRes(termRes);
                        }
                    });
                }
            });
        },

        stopStack() {
            this.processing = true;

            this.$root.emitAgent(this.endpoint, "stopStack", this.stack.name, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
            });
        },

        downStack() {
            this.processing = true;

            this.$root.emitAgent(this.endpoint, "downStack", this.stack.name, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
            });
        },

        restartStack() {
            this.processing = true;

            this.$root.emitAgent(this.endpoint, "restartStack", this.stack.name, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
            });
        },

        updateStack() {
            this.processing = true;

            this.$root.emitAgent(this.endpoint, "updateStack", this.stack.name, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
            });
        },

        deleteDialog() {
            this.$root.emitAgent(this.endpoint, "deleteStack", this.stack.name, (res) => {
                this.$root.toastRes(res);
                if (res.ok) {
                    this.$router.push("/");
                }
            });
        },

        discardStack() {
            this.loadStack();
            this.isEditMode = false;
        },

        highlighterYAML(code) {
            if (!languages.yaml_with_symbols) {
                languages.yaml_with_symbols = languages.insertBefore("yaml", "punctuation", {
                    "symbol": prismjsSymbolDefinition["symbol"]
                });
            }
            return highlight(code, languages.yaml_with_symbols);
        },

        highlighterENV(code) {
            if (!languages.docker_env) {
                languages.docker_env = {
                    "comment": {
                        pattern: /(^#| #).*$/m,
                        greedy: true
                    },
                    "keyword": {
                        pattern: /^\w*(?=[:=])/m,
                        greedy: true
                    },
                    "value": {
                        pattern: /(?<=[:=]).*?((?= #)|$)/m,
                        greedy: true,
                        inside: {
                            "string": [
                                {
                                    pattern: /^ *'.*?(?<!\\)'/m,
                                },
                                {
                                    pattern: /^ *".*?(?<!\\)"|^.*$/m,
                                    inside: prismjsSymbolDefinition
                                },
                            ],
                        },
                    },
                };
            }
            return highlight(code, languages.docker_env);
        },

        enableEditMode() {
            this.isEditMode = true;
        },

        stackNameToLowercase() {
            this.stack.name = this.stack?.name?.toLowerCase();
        },

        onTimestampToggle() {
            // Save preference to localStorage
            localStorage.setItem("showTimestamps", this.showTimestamps.toString());
            
            // Rejoin the terminal with the new setting
            if (!this.isAdd && this.stack.name) {
                // Leave the current terminal
                this.$root.emitAgent(this.endpoint, "leaveCombinedTerminal", this.stack.name, () => {
                    // Rejoin with new timestamp setting
                    this.$root.emitAgent(this.endpoint, "joinCombinedTerminal", this.stack.name, this.showTimestamps, (res) => {
                        if (!res.ok) {
                            this.$root.toastRes(res);
                        }
                    });
                });
            }
        },

    }
};
</script>

<style scoped lang="scss">
@import "../styles/vars.scss";

.terminal {
    height: 200px;
}

.editor-box {
    font-family: 'JetBrains Mono', monospace;
    font-size: 14px;
    &.edit-mode {
        background-color: #2c2f38 !important;
    }
}

.agent-name {
    font-size: 13px;
    color: $dark-font-color3;
}
</style>
