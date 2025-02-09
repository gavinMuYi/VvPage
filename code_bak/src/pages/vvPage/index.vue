<template>
    <div :class="['factory-box', { 'factory-box-errShow': errShow }]">
        <div :class="['factory', { 'errShow': errShow }]">
            <pre class="globel-data" v-if="globelData">{{cusComp}}</pre>
            <div class="save-msg" v-if="dosave">
                <div class="input-box"><input placeholder="页面名称" v-model="pageData.pageName" /></div>
                <div class="input-box"><input placeholder="页面ID" v-model="pageData.metaID" /></div>
                <div class="input-box"><input placeholder="页面备注" v-model="pageData.remark" /></div>
                <div class="btn-bar">
                    <div class="btn" @click="save">保存</div>
                    <div class="btn" @click="dosave = false">取消</div>
                </div>
            </div>
            <div class="top-bar">
                <preScriptEditor ref="preScriptEditorPanel" />
                <div class="pre-script">
                    <div class="btn-xiala" @click="openPreScriptEditor">
                        <span class="iconfont icon-xiala2"></span>
                    </div>
                </div>
                <div class="pro-title">
                    Vv Page<span class="iconfont icon-yezhu"></span>
                </div>
                <interface-editor ref="interface" />
                <div class="actions">
                    <span @click="changeMock">
                        <span class="iconfont icon-quanjituanxiangmuguanli"></span>
                        mock调试{{ windowMock ? 'on' : 'off' }}
                    </span>
                    <span @click="showInterface" class="interface-btn">
                        <span class="iconfont icon-fuzhi"></span>
                        接口协议
                    </span>
                    <span @click="globelData = !globelData">
                        <span class="iconfont icon-xinxitishi"></span>
                        页面信息
                    </span>
                    <span @click="preview = !preview;previewMode.preview = !previewMode.preview;refreshWorkSpace();">
                        <span class="iconfont icon-qiehuan1"></span>
                        {{ preview ? '预览' : '配置' }}态
                    </span>
                    <span @click="abs = !abs;refreshWorkSpace();" class="layout">
                        <span class="iconfont icon-moxingzuzhuang"></span>
                        {{ abs ? '拖动' : '排列' }}
                    </span>
                    <span @click="back" :class="{'disable-action': !snapshot_flag}">
                        <span class="iconfont icon-shangyibu"></span>
                        撤销
                    </span>
                    <span @click="redo" :class="{'disable-action': snapshot_flag === snapshot.length - 1 || !snapshot.length}">
                        <span class="iconfont icon-liuchengtuh5-29"></span>
                        重做
                    </span>
                    <span @click="errShow = !errShow" class="layout erricon">
                        <span class="iconfont icon-tishi1"></span>
                        报错信息
                    </span>
                    <span @click="dosave = true" class="iconLast">
                        <span class="iconfont icon-baocun_mian"></span>
                        保存
                    </span>
                </div>
            </div>
            <div :class="['work-space', {'preview': preview}]">
                <left-bar :preview="preview" @editContent="editContent" />
                <div class="space-content">
                    <div class="component-draw-space" ref="drawSpace" @drop="drop" @dragover="ev => {ev.preventDefault()}">
                        <draw-board
                            :comps="comps"
                            :topDataLevel="topDataLevel"
                            @editComponent="editComponent"
                            @editContent="editContent"
                            @changeInline="changeInline"
                            @action="action"
                            :cusComp="cusComp"
                            :key="refresh"
                            :preview="preview" />
                    </div>
                </div>
                <right-bar
                    :nowEdit="nowEdit"
                    :cusComp="cusComp"
                    :lifeCycle="lifeCycle"
                    @updateParams="updateParams"
                    @globalData="setGlobalData"
                    @lifeCycle="setLifeCycle"
                    @setProps="setProps"
                    @setV="setV"
                    @slotChange="slotChange" />
            </div>
        </div>
        <div id="error-box"></div>
    </div>
</template>

<script>
    import clone from 'clone';
    import qs from 'qs';
    import preScriptEditor from '../../components/preScriptEditor';
    const InterfaceEditor = window.xingine.InterfaceEditor;
    import LeftBar from './components/leftBar';
    import RightBar from './components/rightBar';
    const DrawBoard  = window.xingine.DrawBoard;
    import { iconCompMap, outCompMap } from './config.js';
    import { createHash } from '../../utils/common.js';
    import gtml from '../../G-HTML';
    var cmps = gtml;
    cmps = {
        ...cmps,
        ...window.$Manager('get', 'customerCamps')
    };
    export default {
        name: 'VvPage',
        components: {
            LeftBar,
            RightBar,
            DrawBoard,
            InterfaceEditor,
            preScriptEditor
        },
        provide () {
            return {
                previewMode: this.previewMode
            };
        },
        props: {
            preCusComp: {
                type: Object,
                default: undefined
            },
            topDataLevel: {
                type: Boolean,
                default: true
            }
        },
        data () {
            window.$Manager('set', 'mock', true);
            return {
                errShow: false,
                windowMock: true,
                dosave: false,
                globelData: false,
                abs: true,
                preview: false,
                refresh: 0,
                iconCompMap: iconCompMap,
                outCompMap: outCompMap,
                nowEdit: {},
                rightBarChange: false,
                comps: this.preCusComp ? this.preCusComp.comps : [],
                cusCompHash: this.preCusComp ? this.preCusComp.config.hash : createHash(4),
                config_data_data_bak: this.preCusComp ? this.preCusComp.config.data.data : {},
                config_data_eventHandlers_bak: this.preCusComp ? this.preCusComp.config.data.eventHandlers : {},
                snapshot: [],
                snapshot_flag: 0,
                pageData: {
                    pageName: '',
                    metaID: '',
                    remark: ''
                },
                globalData: {},
                lifeCycle: this.preCusComp ? this.preCusComp.config.lifeCycle : {},
                previewMode: {
                    preview: false
                }
            }
        },
        computed: {
            cusComp () {
                var config = {
                    content: true, // 仅用于配置
                    type: 'combination',
                    config: {
                        hash: this.cusCompHash,
                        data: {
                            props: {},
                            data: {
                                ...this.getCompsProps(this.comps),
                                ...this.config_data_data_bak,
                                ...this.globalData
                            },
                            eventHandlers: {
                                ...this.getCompsEvents(this.comps),
                                ...this.config_data_eventHandlers_bak
                            },
                            emitEvents: []
                        },
                        lifeCycle: {
                            ...this.lifeCycle
                        }
                    },
                    comps: this.comps
                };
                if (this.topDataLevel) {
                    window.GlobelMeta = config;
                }
                return config;
            }
        },
        watch: {
            nowEdit: {
                handler (val) {
                    // this.refreshWorkSpace();
                },
                deep: true
            }
        },
        mounted () {
            this.$set(this, 'nowEdit', this.cusComp);
        },
        methods: {
            save () {
                this.pageData.meta = this.cusComp;
                this.pageData.lastModifiy = +new Date();
                this.$ajax.post('/saveMeta',
                                qs.stringify({ pageData: JSON.stringify(this.pageData) })
                ).then(e => {
                    if (e.code === 0) {
                        alert('保存成功~');
                    }
                });
                this.dosave = false;
            },
            openPreScriptEditor () {
                this.$refs.preScriptEditorPanel.openPannel();
            },
            showInterface () {
                this.$refs.interface.visibleChange();
            },
            changeMock () {
                this.windowMock = !this.windowMock;
                window.$Manager('set', 'mock', !window.$Manager('get', 'mock'));
            },
            slotChange (slotData) {
                this.refreshWorkSpace();
                this.comps.forEach(item => {
                    if (item.config.hash === slotData.hash) {
                        item.config.slot.forEach(slot => {
                            if (slot.name === slotData.slotName) {
                                this.$set(slot, 'children', slotData.slots);
                            }
                        });
                    }
                });
                this.$set(this, 'config_data_data_bak', slotData.data);
                this.$set(this, 'config_data_eventHandlers_bak', slotData.event);
            },
            setProps (hash, datakey, funcStr, mode) {
                this.comps.forEach(item => {
                    if (item.config.hash === hash) {
                        if (item.config.props[datakey] && !mode) {
                            this.$set(item.config.props, datakey, '');
                            delete item.config.props[datakey];
                        } else {
                            this.$set(item.config.props, datakey, funcStr);
                        }
                    }
                });
            },
            setV (hash, func, key) {
                var funcStr = key === 'vif'
                    ? 'function () {return true;}'
                    : 'function () {return "";}';
                this.comps.forEach(item => {
                    if (item.config.hash === hash) {
                        if (!func) {
                            if (!item.config[key]) {
                                item.config[key] = funcStr;
                            } else {
                                item.config[key] = func;
                            }
                        } else {
                            item.config[key] = func;
                        }
                    }
                });
            },
            action (action, hash) {
                let copyHash = createHash(4);
                let copyItem = {};
                let configString = '';
                switch (action) {
                case 'delete':
                    this.comps.forEach((item, i) => {
                        item.config.hash === hash && this.comps.splice(i, 1);
                    });
                    Object.keys(this.config_data_data_bak).forEach(key => {
                        key.indexOf(hash) > -1 && (delete this.config_data_data_bak[key]);
                    });
                    Object.keys(this.config_data_eventHandlers_bak).forEach(key => {
                        key.indexOf(hash) > -1 && (delete this.config_data_data_bak[key]);
                    });
                    break;
                case 'copy':
                    this.comps.forEach((item, i) => {
                        if (item.config.hash === hash) {
                            copyItem = clone(item);
                            configString = JSON.stringify(copyItem.config);
                            configString = configString.replace(new RegExp(hash, 'gm'), copyHash);
                            copyItem.config = JSON.parse(configString);
                            this.comps.push(copyItem);
                        }
                    });
                    Object.keys(this.config_data_data_bak).forEach(key => {
                        key.indexOf(hash) > -1 && (this.config_data_data_bak[copyHash] = clone(this.config_data_data_bak[key]));
                    });
                    Object.keys(this.config_data_eventHandlers_bak).forEach(key => {
                        key.indexOf(hash) > -1 && (this.config_data_data_bak[key.replace(hash, copyHash)] = clone(this.config_data_data_bak[key]));
                    });
                    break;
                }
            },
            refreshWorkSpace () {
                this.$nextTick(() => {
                    if (this.snapshot_flag === this.snapshot.length - 1 || !this.snapshot.length) {
                        if (!this.snapshot[0]) {
                            this.snapshot.push(clone(this.cusComp));
                            this.snapshot_flag = 0;
                        } else {
                            if (JSON.stringify(this.snapshot[this.snapshot.length - 1]) !== JSON.stringify(this.cusComp)) {
                                this.snapshot.push(clone(this.cusComp));
                                this.snapshot_flag++;
                            }
                        }
                    } else if (this.snapshot_flag < this.snapshot.length - 1) {
                        if (
                            JSON.stringify(this.snapshot[this.snapshot_flag + 1]) !== JSON.stringify(this.cusComp) &&
                            JSON.stringify(this.snapshot[this.snapshot_flag]) !== JSON.stringify(this.cusComp)
                        ) {
                            this.snapshot = this.snapshot.slice(0, this.snapshot_flag + 1);
                            if (JSON.stringify(this.snapshot[this.snapshot.length - 1]) !== JSON.stringify(this.cusComp)) {
                                this.snapshot.push(clone(this.cusComp));
                                this.snapshot_flag++;
                            }
                        }
                    }
                    this.refresh++;
                });
            },
            back () {
                if (!this.snapshot_flag) {
                    return;
                }
                this.snapshot_flag--;
                this.updateParams(clone(this.snapshot[this.snapshot_flag]));
                this.$set(this, 'comps', clone(this.snapshot[this.snapshot_flag].comps));
                this.refreshWorkSpace();
            },
            redo () {
                if (this.snapshot_flag === this.snapshot.length - 1 || !this.snapshot.length) {
                    return;
                }
                this.snapshot_flag++;
                this.updateParams(clone(this.snapshot[this.snapshot_flag]));
                this.$set(this, 'comps', clone(this.snapshot[this.snapshot_flag].comps));
                this.refreshWorkSpace();
            },
            setGlobalData (val) {
                this.$set(this, 'globalData', JSON.parse(val));
            },
            setLifeCycle (key, val) {
                this.$set(this.lifeCycle, key, val);
            },
            updateParams (comps) {
                if (this.rightBarChange) {
                    this.rightBarChange = false;
                    return;
                }
                this.refreshWorkSpace();
                if (comps.content) {
                    this.config_data_data_bak = comps.config.data.data;
                    this.config_data_eventHandlers_bak = comps.config.data.eventHandlers;
                } else {
                    this.config_data_data_bak = this.cusComp.config.data.data;
                    this.config_data_eventHandlers_bak = this.cusComp.config.data.eventHandlers;
                }
            },
            getCompsEvents (comps) {
                this.refreshWorkSpace();
                let events = {};
                comps.forEach(comp => {
                    cmps[comp.name].event && cmps[comp.name].event.forEach(func => {
                        this.$set(events, comp.config.hash + '-' + func.name, {
                            name: func.name,
                            label: func.label,
                            params: func.params,
                            handler: `function ${func.name + comp.config.hash}() {}`
                        });
                    });
                });
                return events;
            },
            getCompsProps (comps) {
                this.refreshWorkSpace();
                let props = {};
                comps.forEach(comp => {
                    this.$set(props, comp.config.hash, {});
                    var prop = cmps[comp.name].props;
                    for (let key in prop) {
                        let value;
                        if (!(prop[key].type instanceof Array) && prop[key].type) {
                            value = typeof prop[key].type() === 'object' ? JSON.stringify(prop[key].default()) : prop[key].default;
                        }
                        this.$set(props[comp.config.hash], key, value);
                    }
                });
                return props;
            },
            changeInline (prehash, hash) {
                let preId = 0;
                let id = 0;
                this.comps.forEach((item, i) => {
                    item.config.hash === prehash && (preId = i);
                    item.config.hash === hash && (id = i);
                });
                let item = this.comps.splice(preId, 1);
                this.comps.splice(id, 0, item[0]);
            },
            drop (ev) {
                ev.preventDefault();
                var data = JSON.parse(ev.dataTransfer.getData('DragComp'));
                var domabs = (data.x + data.y) || (data.x + data.y === 0);
                var dragCompData = {
                    ...data,
                    id: data.id,
                    x: this.abs || domabs ? ev.clientX - this.$refs.drawSpace.offsetLeft : undefined,
                    y: this.abs || domabs ? ev.clientY - this.$refs.drawSpace.offsetTop : undefined,
                    name: this.iconCompMap[data.id] || this.outCompMap[data.id]
                };
                if (data.config && data.config.hash && !domabs) {
                    dragCompData.x = undefined;
                    dragCompData.y = undefined;
                    return;
                }
                data.config !== undefined
                    ? (dragCompData.config = data.config)
                    : (dragCompData.config = {
                        hash: createHash(4),
                        props: {},
                        directives: 'function () { return []; }',
                        slot: (cmps[this.iconCompMap[data.id] || this.outCompMap[data.id]].slot || []).map(e => { return { ...e, children: [] } }),
                        vif: undefined,
                        vfor: undefined
                });
                data.index !== undefined && this.comps.splice(data.index, 1);
                if (cmps[dragCompData.name].cusDirectives && cmps[dragCompData.name].cusDirectives.length) {
                    this.comps.unshift(dragCompData);
                } else {
                    this.comps.push(dragCompData);
                }
                this.editComponent(dragCompData);
            },
            editComponent (e) {
                if (JSON.stringify(this.nowEdit) !== JSON.stringify(e)) {
                    this.$set(this, 'nowEdit', e);
                    this.rightBarChange = true;
                }
            },
            editContent () {
                if (JSON.stringify(this.nowEdit) !== JSON.stringify(this.cusComp)) {
                    this.$set(this, 'nowEdit', this.cusComp);
                    this.rightBarChange = true;
                }
            }
        }
    }
</script>

<style lang="less">
.factory-box {
    height: 100%;
    overflow: hidden;
    width: 100%;
    #error-box {
        width: 350px;
        height: 100%;
        display: inline-block;
        background: #000202d9;
        color: #fff;
        font-size: 18px;
        font-weight: 600;
        vertical-align: top;
        overflow: auto;
        text-indent: 2em;
        padding: 10px;
        box-sizing: border-box;
        div {
            margin-bottom: 10px;
        }
    }
    .errShow {
        width: ~'calc(100% - 350px)';
        display: inline-block;
        #stylePanelOther,
        #stylePanelContent,
        #stylePanel {
            width: ~'calc(100% - 350px)'!important;
        }
    }
}
.factory-box-errShow {
    min-width: 1850px;
    font-size: 0px;
}
.factory {
    height: 100%;
    overflow: hidden;
    font-size: 16px;
    .save-msg {
        z-index: 100000;
        position: absolute;
        width: 400px;
        height: 300px;
        top: 50%;
        left: 50%;
        margin-left: -200px;
        padding: 20px;
        margin-top: -150px;
        border: 1px solid @main;
        background: @mainop;
        border-radius: 10px;
        text-align: center;
        .btn-bar {
            margin-top: 30px;
            .btn {
                display: inline-block;
                width: 80px;
                vertical-align: middle;
                height: 26px;
                line-height: 26px;
                display: inline-block;
                text-align: center;
                font-size: 14px;
                padding: 2px 5px;
                border-radius: 3px;
                background: #ededed;
                margin: 20px;
            }
        }
        .input-box {
            height: 40px;
            margin: 20px auto;
            width: 250px;
            box-sizing: border-box;
            padding: 2px 8px;
            border: 1px solid #dadee7;
            border-radius: 2px;
            transition: border-color .25s;
            input {
                position: relative;
                display: block;
                width: 100%;
                line-height: inherit;
                height: 34px;
                padding: 7px 0;
                color: #1e2330;
                outline: none;
                border: 0 none;
                background: none;
                box-sizing: border-box;
                width: 100%;
            }
        }
    }
    .globel-data {
        position: fixed;
        top: 0;
        left: 0;
        background: @mainop;
        color: @dep;
        width: 800px;
        max-height: 800px;
        overflow: auto;
        margin: 0;
        z-index: 1000000;
    }
    .top-bar {
        height: 100px;
        // background: #f7f8fa;
        background: @main;
        // border-bottom: 1px solid #ededed;
        box-sizing: border-box;
        .disable-action {
            color: #cabcbc;
            &:hover {
                cursor: not-allowed;
            }
        }
        .pre-script {
            position: fixed;
            top: 0;
            left: 50%;
            .btn-xiala {
                width: 50px;
                height: 50px;
                border-radius: 50%;
                vertical-align: middle;
                text-align: center;
                line-height: 50px;
                background: #0a403a;
                position: relative;
                top: -25px;
                left: -25px;
                font-weight: 700;
                .iconfont {
                    color: @dep;
                }
                &:hover {
                    top: -10px;
                    cursor: pointer;
                }
            }
        }
        .pro-title {
            line-height: 100px;
            font-size: 32px;
            font-weight: 700;
            color: @dep!important;
            margin-left: 30px;
            display: inline-block;
            .icon-yezhu {
                font-size: 45px;
                display: inline-block;
                padding-left: 14px;
                line-height: 100px;
                position: relative;
                top: 0px;
                left: 5px;
                color: @dep!important;
                transform: rotateY(180deg);
            }
        }
        .actions {
            float: right;
            color: @dep;
            font-weight: 700;
            user-select: none;
            .iconfont {
                padding-left: 5px;
            }
            .icon-baocun_mian {
                font-size: 20px;
                line-height: 100px;
                color: @dep;
                font-weight: 400;
                padding-left: 5px;
                position: relative;
                top: 2px;
            }
            .iconLast {
                margin-right: 30px;
            }
        }
    }
    // .left-bar,
    // .right-bar {
    //     background: @main;
    // }
    .work-space {
        height: ~'calc(100% - 100px)';
        font-size: 0px;
        .space-content {
            height: 100%;
            width: ~'calc(100% - 620px)';
            display: inline-block;
            vertical-align: top;
            background: #ededed;
            padding: 10px;
            box-sizing: border-box;
        }
        .component-draw-space {
            display: inline-block;
            position: relative;
            height: 100%;
            background: #fff;
            width: 100%;
            vertical-align: top;
            font-size: 12px;
            overflow: auto;
            box-shadow: 5px 4px 1px #88888830;
        }
    }
    .preview {
        .left-bar,
        .right-bar {
            position: relative;
            &:before {
                content: '';
                position: absolute;
                background: #011210e8;
                opacity: 0.7;
                z-index: 10000;
                width: 100%;
                height: 100%;
                &:hover {
                    cursor: not-allowed;
                }
            }
            &:hover {
                cursor: not-allowed;
            }
        }
        .right-bar {
            &:before {
                width: 310px;
            }
        }
    }
    .title {
        width: 295px;
        line-height: 30px;
        font-size: 14px;
        font-weight: 700;
        padding-left: 15px;
        background: @sub;
        color: @dep;
    }
    .materials-list {
        width: 310px;
        font-size: 0;
        padding: 0;
        overflow: auto;
        .material {
            display: inline-block;
            vertical-align: top;
            font-size: 12px;
            letter-spacing: normal;
            word-spacing: normal;
            line-height: inherit;
            width: 60px;
            padding-bottom: 10px;
            text-align: center;
            list-style: none !important;
            cursor: pointer;
            .list-icon {
                display: block;
                height: 70px;
                line-height: 70px;
                font-size: 24px;
                margin: 0 auto 5px;
                color: @main;
                -webkit-transition: font-size 0.15s linear, width 0.15s linear;
                -moz-transition: font-size 0.15s linear, width 0.15s linear;
                transition: font-size 0.15s linear, width 0.15s linear;
                &:hover {
                    font-size: 56px;
                    color: @sub;
                }
            }
            .material-name {
                color: @main;
            }
        }
    }
}
</style>
