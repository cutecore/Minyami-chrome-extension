<template>
    <div class="main-container">
        <div>
            <div class="buttons">
                <el-button type="text" @click="showConfig = true">{{ $t("message.settings") }}</el-button>
                <el-button type="text" @click="changeLanguage">
                    {{ $i18n.locale === "en" ? "中文" : "English" }}
                </el-button>
            </div>
            <el-card v-if="showConfig" class="config-container" shadow="never">
                <el-form>
                    <el-form-item :label="$t('message.threads')">
                        <el-input v-model="configForm.threads"></el-input>
                    </el-form-item>
                    <el-form-item>
                        <el-button @click="saveConfig">{{ $t("message.save") }}</el-button>
                    </el-form-item>
                </el-form>
            </el-card>
        </div>
        <el-card v-if="playlists.length === 0" shadow="never">
            <div style="text-align: center">{{ $t("message.noData") }}</div>
        </el-card>
        <el-card v-if="showNoKeyWarning()" shadow="never">
            <el-alert :title="$t('message.noKeyWarning')" type="error"></el-alert>
        </el-card>
        <el-card v-if="showNoCookiesWarning()" shadow="never">
            <el-alert :title="$t('message.noCookieWarning')" type="error"></el-alert>
        </el-card>
        <el-card v-if="showCookieWarning()" shadow="never">
            <el-alert :title="$t('message.cookieWarning')" type="warning"></el-alert>
        </el-card>
        <el-card v-if="showNotSupported()" shadow="never">
            <el-alert :title="$t('message.unsupportedTip')" type="error"></el-alert>
        </el-card>
        <el-card class="playlist-item" v-for="playlist in playlists" :key="playlist.url" shadow="never">
            <div>
                <div :title="playlist.url" class="playlist-item-playlist-url">
                    <span>{{ playlist.url }}</span>
                </div>
                <el-form :inline="true">
                    <el-form-item>
                        <el-checkbox v-model="form.recoverMode" :label="$t('message.resumeMode')"></el-checkbox>
                    </el-form-item>
                    <el-form-item>
                        <el-checkbox v-model="form.live" :label="$t('message.liveMode')"></el-checkbox>
                    </el-form-item>
                </el-form>
                <template v-for="chunkList in playlist.chunkLists">
                    <el-form class="playlist-chunklist-item" :key="chunkList.url">
                        <el-form-item>
                            <div class="playlist-chunklist-item-info">
                                <template v-if="chunkList.minimumMinyamiVersion">
                                    <el-tag type="info" size="mini" class="playlist-tag">
                                        {{ $t("message.require") }} Minyami >= {{ chunkList.minimumMinyamiVersion }}
                                    </el-tag>
                                </template>
                                <template v-if="chunkList.type === 'video'">
                                    <el-tag type="info" size="mini" class="playlist-tag">
                                        {{ $t("message.video") }}
                                    </el-tag>
                                    <span v-if="chunkList.resolution">
                                        {{ $t("message.resolution") }}: {{ chunkList.resolution.x }} ×
                                        {{ chunkList.resolution.y }}
                                    </span>
                                    <span v-if="chunkList.bandwidth">
                                        {{ $t("message.bitrate") }}:
                                        {{ Math.round(chunkList.bandwidth / 1024) }}
                                        kbps
                                    </span>
                                </template>
                                <template v-if="chunkList.type === 'audio'">
                                    <el-tag type="info" size="mini" class="playlist-tag">
                                        {{ $t("message.audio") }}
                                    </el-tag>
                                    <span>
                                        {{ chunkList.name && ` ${$t("message.trackName")}: ${chunkList.name}` }}
                                    </span>
                                </template>
                            </div>
                        </el-form-item>
                        <el-form-item class="playlist-chunklist-command">
                            <el-input
                                size="mini"
                                :value="generateCommand(chunkList, playlist)"
                                :ref="chunkList.url"
                            ></el-input>
                        </el-form-item>
                        <el-form-item>
                            <div>
                                <el-button size="small" @click="copy(chunkList)">{{ $t("message.copy") }}</el-button>
                            </div>
                        </el-form-item>
                    </el-form>
                </template>
            </div>
        </el-card>
    </div>
</template>
<style>
.main-container {
    min-width: 600px;
    min-height: 600px;
}
.buttons {
    display: flex;
    justify-content: space-between;
}
.config-container {
    margin: 1rem auto;
}
.playlist-item {
    margin: 1rem 0;
    max-width: 600px;
}
.playlist-item-playlist-url {
    text-overflow: ellipsis;
    overflow: hidden;
    word-wrap: none;
    white-space: nowrap;
}
.playlist-item .el-form-item {
    margin-bottom: 0 !important;
}
.playlist-tag {
    margin-right: 1rem;
}
.el-card {
    margin: 10px auto;
}
</style>
<script>
import Storage from "../../core/utils/storage.js";
import { supportedSites } from "../../definitions";
const needCookiesSites = ["360ch.tv"];
const needKeySites = ["abema.tv", "live2.nicovideo.jp", "dmm.com", "dmm.co.jp", "hibiki-radio.jp"];
export default {
    data() {
        return {
            playlists: [],
            keys: [],
            cookies: [],
            form: {
                recoverMode: false,
                live: false
            },
            configForm: {
                threads: ""
            },
            currentUrl: "",
            showConfig: false
        };
    },
    mounted() {
        this.configForm.threads = Storage.getConfig("threads");
        setInterval(this.getKeys, 1000);
        setInterval(this.getCookies, 1000);
        setInterval(this.check, 1000);
        this.check();
        this.getKeys();
        this.getCookies();
    },
    methods: {
        check() {
            chrome.tabs.query({ active: true, currentWindow: true }, tabs => {
                if (tabs[0]) {
                    this.playlists = Storage.getHistory(tabs[0].url);
                    this.currentUrl = tabs[0].url;
                }
            });
        },
        generateCommand(chunklist, playlist) {
            const prefix = "minyami";
            let command = "";
            if (!this.form.recoverMode) {
                command += `${prefix} -d "${chunklist.url}" --output "${playlist.title.replace(/\"/g, `\\"`)}.ts"`;
            } else {
                command += `${prefix} -r "${chunklist.url}"`;
            }
            if (this.keys.length > 0) {
                command += ` --key "${this.keys[0]}"`;
            }
            if (this.cookies.length > 0) {
                command += ` --headers "Cookie: ${this.cookies[0]}"`;
            }
            if (this.form.live) {
                command += ` --live`;
            }
            if (Storage.getConfig("threads")) {
                command += ` --threads ${Storage.getConfig("threads")}`;
            }
            return command;
        },
        copy(chunkList) {
            const input = this.$refs[chunkList.url];
            input[0].select();
            document.execCommand("copy");
        },
        getKeys() {
            chrome.tabs.query({ active: true, currentWindow: true }, tabs => {
                if (tabs[0]) {
                    const keys = Storage.getHistory(tabs[0].url + "-key");
                    if (keys && keys.length > 0) {
                        this.keys = keys;
                    }
                }
            });
        },
        getCookies() {
            chrome.tabs.query({ active: true, currentWindow: true }, tabs => {
                if (tabs[0]) {
                    const cookies = Storage.getHistory(tabs[0].url + "-cookies");
                    if (cookies && cookies.length > 0) {
                        this.cookies = cookies;
                    }
                }
            });
        },
        showNoKeyWarning() {
            if (!this.currentUrl) {
                return false;
            }
            for (const site of needKeySites) {
                if (this.currentUrl.includes(site) && this.keys.length === 0) {
                    return true;
                }
            }
            return false;
        },
        showNoCookiesWarning() {
            if (!this.currentUrl) {
                return false;
            }
            for (const site of needCookiesSites) {
                if (this.currentUrl.includes(site) && this.cookies.length === 0) {
                    return true;
                }
            }
            return false;
        },
        showCookieWarning() {
            if (!this.currentUrl) {
                return false;
            }
            for (const site of needCookiesSites) {
                if (this.currentUrl.includes(site) && this.cookies.length === 0) {
                    return true;
                }
            }
            return false;
        },
        showNotSupported() {
            return !this.currentUrl || !supportedSites.includes(new URL(this.currentUrl).host);
        },
        saveConfig() {
            const threads = this.configForm.threads;
            Storage.setConfig("threads", threads);
            this.showConfig = false;
        },
        changeLanguage() {
            const targetLanguage = this.$i18n.locale === "en" ? "zh_CN" : "en";
            Storage.setConfig("language", targetLanguage);
            this.$i18n.locale = targetLanguage;
        }
    }
};
</script>
