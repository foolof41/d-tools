<template>
    <div>
        <div ref="hostEdit" class="host-edit">
            <el-tabs v-model="activeName">
                <el-tab-pane :label="$t('textMode')" name="text">
                    <div class="text-edit">
                        <textarea v-model="hostsText"></textarea>
                        <div class="btn-container">
                            <el-button type="success" @click="saveText">{{$t('save')}}</el-button>
                        </div>
                    </div>
                </el-tab-pane>
                <el-tab-pane :label="$t('tableMode')" name="table">
                    <div class="table-edit">
                        <template v-if="hostsCount<30">
                            <el-table :data="localHosts" stripe :show-header="false">
                                <el-table-column min-width="27">
                                    <template slot-scope="scope">
                                        <el-input size="mini" type="text" v-model="scope.row.ip"></el-input>
                                    </template>
                                </el-table-column>
                                <el-table-column min-width="30">
                                    <template slot-scope="scope">
                                        <el-input size="mini" type="text" v-model="scope.row.domain"></el-input>
                                    </template>
                                </el-table-column>
                                <el-table-column min-width="10">
                                    <template slot-scope="scope">
                                        <el-button class="btn-remove" size="mini" type="danger" icon="el-icon-delete"
                                                   circle @click="removeRow(scope.$index)"></el-button>
                                    </template>
                                </el-table-column>
                            </el-table>
                            <el-button class="btn-add" size="mini" type="success" icon="el-icon-plus" circle
                                       @click="addRow"></el-button>
                            <div class="btn-container">
                                <el-button type="success" @click="saveTable">{{$t('save')}}</el-button>
                            </div>
                        </template>
                        <template v-else>
                            <div class="too-many-info">
                                {{$t('tooLarge')}}
                            </div>
                        </template>
                    </div>
                </el-tab-pane>
                <el-tab-pane label="ping" name="ping">
                    <div class="ping-edit">
                        <el-input :placeholder="$t('plsInput')" size="small" class="ip-input"
                                  v-model="pingIp"></el-input>
                        <el-button class="btn-add" size="small " type="success" @click="startPing" v-if="!pingStatus">
                            Ping
                        </el-button>
                        <el-button class="btn-add" size="small " type="danger" @click="stopPing" v-if="pingStatus">Stop
                        </el-button>
                        <textarea class="ping-result" ref="textarea">
                                {{pingResult}}
                        </textarea>
                    </div>
                </el-tab-pane>
            </el-tabs>
        </div>
    </div>
</template>

<i18n>
    {
    "en_US": {
    "textMode": "Text mode",
    "tableMode": "Table mode",
    "save":"Save",
    "tooLarge":"The number of hosts files is too large (> 30), and the table mode has been disabled",
    "plsInput":"IP or domain name",
    "saveSuccess":"Save success",
    "saveFail":"Save fail, Permission denied"
    },

    "zh_CN": {
    "textMode": "文本模式",
    "tableMode": "表格模式",
    "save":"保存",
    "tooLarge":"当前 hosts 文件条数过多( ≥30 ), 已禁用表格模式",
    "plsInput":"请输入IP或域名",
    "saveSuccess":"保存成功",
    "saveFail":"保存失败, 当前用户无 hosts 修改权限"
    }}
</i18n>

<script>
    import fs from 'fs';
    import os from 'os';
    import readline from 'readline';
    import {Loading} from 'element-ui';
    import {exec} from 'child_process';
    import kill from 'tree-kill';
    import {copyToClipboard} from '@/utils';

    const sys_hosts_path = process.platform === 'win32' ? `${process.env.windir || 'C:\\WINDOWS'}\\system32\\drivers\\etc\\hosts` : '/etc/hosts';

    export default {
        name: 'hostEdit',
        data() {
            return {
                activeName: "text",
                hostsCount: 0,
                hostsText: "",
                localHosts: [],
                pingIp: "",
                pingResult: "",
                pingProcess: "",
                pingStatus: false
            }
        },
        watch: {
            pingResult: function () {
                this.$refs.textarea.scrollTop = this.$refs.textarea.scrollHeight;
            }
        },
        methods: {
            startPing() {
                this.pingResult = "";
                this.pingStatus = true;
                if (this.pingIp.trim() != "") {
                    if (process.platform === 'darwin' || process.platform === 'linux') {
                        this.pingProcess = exec(`ping ${this.pingIp}`, {encoding: 'UTF-8'});
                    } else {
                        this.pingProcess = exec(`@chcp 65001 >nul & ping ${this.pingIp} -t`, {encoding: 'UTF-8'});
                    }
                    this.pingProcess.stdout.on('data', (data) => {
                        this.pingResult += data;
                    });
                    this.pingProcess.on('close', (code) => {
                        this.pingStatus = false;
                        this.pingResult += `Exit  code:${code}`;
                    });
                }
            },
            stopPing() {
                kill(this.pingProcess.pid);
            },
            removeRow(i) {
                this.localHosts.splice(i, 1);
            },
            addRow() {
                this.localHosts.push({ip: "", domain: "", error: false});
            },
            saveTable() {
                let flag = true;
                this.localHosts.forEach((item) => {
                    if (item.ip.trim() == "" || item.domain.trim() == "") {
                        flag = false;
                        item.error = true;
                    } else {
                        item.error = false;
                    }
                });
                if (flag) {
                    let str = '';
                    this.localHosts.forEach((item) => {
                        str += `${item.ip} ${item.domain}${os.EOL}`;
                    });
                    this.save(str);
                }
            },
            saveText() {
                const replace = this.hostsText.replace(/\n/g, os.EOL);
                this.save(replace);
            },
            save(str) {
                const loading = Loading.service({fullscreen: true});
                fs.writeFile(sys_hosts_path, str, (err) => {
                    loading.close();
                    if (err) {
                        this.$message({message: `${this.$t('saveFail')}`, type: 'error'});
                    } else {
                        this.$message({message: `${this.$t('saveSuccess')}`, type: 'success'});
                        this.refreshHosts();
                    }
                });
            },
            refreshHosts() {
                this.hostsText = "";
                this.localHosts = [];
                this.hostsCount = 0;
                const lineReader = readline.createInterface({
                    input: fs.createReadStream(sys_hosts_path),
                    crlfDelay: Infinity
                });
                lineReader.on('line', (line) => {
                    this.hostsText += line + '\n';
                    line = this.filterAnnotation(line);
                    if (line != null && line.trim() != "") {
                        this.hostsCount++;
                        if (this.hostsCount < 30) {
                            this.analyseHost(line);
                        }
                    }
                })
            },
            filterAnnotation(text) {
                if (text != null && text.indexOf("#") != -1) {
                    text = text.slice(0, text.indexOf("#"))
                }
                return text;
            },
            analyseHost(text) {
                const textArr = this.trimArraySpace(text.replace(/\t/g, " ").split(" "));
                if (textArr.length >= 2) {
                    this.localHosts.push({ip: textArr[0], domain: textArr[1], error: false});
                }
            },
            trimArraySpace(array) {
                for (var i = 0; i < array.length; i++) {
                    if (array[i] == "" || typeof(array[i]) == "undefined") {
                        array.splice(i, 1);
                        i = i - 1;
                    }
                }
                return array;
            }
        },
        mounted() {
            this.refreshHosts();
            this.$refs.hostEdit.querySelectorAll(".el-input__inner,textarea").forEach((item) => {
                item.addEventListener("dblclick", (e) => {
                    copyToClipboard(e.target.value, this);
                });
            });
        }
    }
</script>

<style lang="less" scoped>
    .host-edit {
        padding: 0px 66px;
        max-width: 900px;
        margin: 0 auto;
    }

    .text-edit {
        width: 100%;
        height: 100%;
        padding-bottom: 30px;

        textarea {
            padding: 5px;
            margin: 0px;
            width: 100%;
            height: calc(~'100vh - 300px');
            min-height: 400px;
            white-space: nowrap;
            font-size: 12px;
            border: 1px dashed #409EFF;
            border-radius: 6px;
        }

        .btn-container {
            text-align: center;
            margin-top: 20px;
        }
    }

    .table-edit {
        width: 100%;
        height: 100%;
        padding-bottom: 30px;

        .too-many-info {
            text-align: center;
            font-size: 14px;
            margin-top: 5px;
        }

        .btn-add, .btn-remove {
            float: right;
        }

        .btn-add {
            margin: 10px;
        }

        .btn-container {
            text-align: center;
            margin-top: 50px;
        }
    }

    .ping-edit {

        .ip-input {
            display: inline-block;
            width: 150px;
            margin-bottom: 15px;
        }

        .ping-result {
            width: 100%;
            height: calc(100vh - 300px);
            min-height: 400px;
            font-size: 14px;
            overflow: auto;
            padding: 5px;
            font-size: 12px;
            border: 1px dashed #409EFF;
            border-radius: 6px;
        }
    }
</style>