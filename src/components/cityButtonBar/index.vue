<template>
    <div class="cityButtonBar filter">
        <div class="bar barWrap"
             v-if="pageState !== 'detail'">
            <el-button @click="reverseDialog"
                       type="primary">添加城市
            </el-button>
            <div class="alignSelf">点击已选择城市</div>
            <el-upload class="upload-demo"
                       action="/coupon/mis/importcsv"
                       :show-file-list="false"
                       :on-success="handleUploadSuccess">
                <el-button type="primary">点击上传
                </el-button>
            </el-upload>
            <div class="alignSelf">支持txt、csv格式，最大5M</div>
            <a href="https://s.waimai.baidu.com/c/static/mis/城市上传模板.csv"
               class="city-demo-download">城市模板下载
            </a>
        </div>
        <div class="">
            <show-items :items="selectedCityInfos"
                        :disabled="pageState === 'detail'"
                        @change="onChangeSelfSelectedCityIds"></show-items>
        </div>
        <el-dialog :visible.sync="dialogVisible"
                   title="请选择城市"
                   size="large">
            <div class="cityDialog once"
                 v-show="showSelectContent">
                <div class="bindCityBox"></div>
                <div class="cityWrap">
                    <div class="city-selector-box">
                        <div class="city-selector-tab-box">
                            <template v-for="(item,$index) in citysGroup">
                                <a @click="activeIndex = $index"
                                   :class="$index === activeIndex ? 'active' : ''"
                                   href="javascript:;">{{item.ind.join('')}}
                                </a>
                            </template>
                            <button type="button"
                                    class="btn btn-default select-all-city-checkbox"
                                    @click="selectCity('1')">全选
                            </button>
                            <button type="button"
                                    class="btn btn-default select-others-city-checkbox"
                                    @click="reverseSelect">反选
                            </button>
                        </div>
                        <template v-for="(item,$index) in citysGroup">
                            <div class="city-selector-content-box"
                                 :class="$index === activeIndex ? 'active' : ''">
                                <template v-for="(citys,key) in item.citys">
                                    <div class="city-selector-group-box">
                                        <div>{{key}}</div>
                                        <template v-for="city in citys">
                                    <span :class="cityStateClass(city.value)"
                                          @click="selectCity(city.value)">
                                        <i></i>
                                         {{city.name}}</span>
                                        </template>
                                    </div>
                                </template>
                            </div>
                        </template>
                    </div>
                </div>
            </div>
        </el-dialog>
    </div>
</template>

<script>
    import axios from 'axios'
    const cityListUrl = '/huodong/mis/transmit?url=%2F%3Fqt%3Dgetcitylist%26type%3D1';
    import cityList from './city.js'
    import transformData from './transformData'
    import _ from 'lodash'
    import showItems from '@/components/showItems/index.vue'
    export default {
        name: 'cityButtonBar',
        props: {
            /*selectedCityIds: {
             type: Array,
             default: function () {
             return []
             }
             },*/
            disabled: {
                type: Boolean,
                default: false
            },
            showSelectContent: {
                type: Boolean,
                default: true
            },
            changeSelectedCityIds: {
                type: Function
            }
        },
        components: {
            showItems: showItems
        },
        computed: {
            pageState: function () {
                let $engine = this.$engine;
                if ($engine && $engine.globalObj) {
                    return $engine.globalObj.pageState
                }
                return ''
            },
            selfShowSelectContent: function () {
                if (this.disabled) {
                    return false;
                }
                return this.showSelectContent;
            },
            selectedCityInfos: function () {
                return this.getCityInfo(this.selfSelectedCityIds)
            },
            countyWideSelected: function () {
                return this.isCountyWide(this.selfSelectedCityIds[0]);
            },
            selectedCityIds: function () {
                return this.selfSelectedCityIds
            }
        },
        watch: {
            selectedCityIds: {
                deep: true,
                immediate: true,
                handler(v){
                    this.selfSelectedCityIds = v
                }
            },
            selfSelectedCityIds: {
                deep: true,
                handler(v){
                    if (v === this.selfSelectedCityIds) return;
                    this.$emit('change', v, this.getCityInfo(v))
                }
            }
        },
        created: function () {
            var me = this;
            this.getCityData(function (citysGroup, citys, cityIds) {
                me.citysGroup = citysGroup;
                me.citys = citys;
                me.cityIds = cityIds;
            })
        },
        methods: {
            reverseDialog(){
                this.dialogVisible = !this.dialogVisible
            },
            onChangeSelfSelectedCityIds(v){
                let i = 0;
                // let originLength = this.selfSelectedCityIds.length
                // for (; i < originLength; i++) {
                //     this.selfSelectedCityIds.pop();
                // }
                // for (i = 0; i < v.length; i++) {
                //     this.selfSelectedCityIds.push(v.unshift());
                //     this.selfSelectedCityIds = this.selfSelectedCityIds;
                // }
                this.selfSelectedCityIds = v;
                if (this.changeSelectedCityIds) {
                    this.changeSelectedCityIds(this.selfSelectedCityIds);
                }
            },
            getCityInfo: function (v) {
                var selectedCitys = [];
                var me = this;
                v.forEach(function (item, index) {
                    selectedCitys[index] = me.getDataByCityId(item);
                });
                return selectedCitys
            },
            getDataByCityId: function (id) {
                return this.citys[id];
            },
            getCityData: function (callback) {
                var me = this;
                if (window.inBlockCanvas) {
                    var result = transformData.cityData(cityList)
                    callback.apply(this, result)
                } else {
                    axios({
                        url: '/marketing/mis/listformatrolecity',
                        type: 'GET',
                        dataType: 'json',
                        async: false,
                    }).then(function (ret) {
                        if (ret && ret.data) {
                            var result = transformData.cityData(ret.data)
                            callback.apply(this, result)
                        }
                    }).catch(function () {
                        var result = transformData.cityData(cityList)
                        callback.apply(this, result)
                    });
                }
            },
            removeSelectedCity: function (city) {
                this.selfSelectedCityIds.splice(_.indexOf(this.selfSelectedCityIds, city), 1);
            },
            cityStateClass: function (city) {
                var isCountyWide = this.isCountyWide(city);
                if (isCountyWide && this.countyWideSelected) {
                    return 'checked';
                } else if (!isCountyWide && this.countyWideSelected) {
                    return 'disabled'
                } else if (this.inSelectedCitys(city)) {
                    return 'checked'
                } else {
                    return ''
                }
            },
            isCountyWide: function (city) {
                if (city) {
                    return city === '1'
                }
            },
            reverseSelect: function () {
                if (this.selfSelectedCityIds[0] === '1') {
                    this.selfSelectedCityIds = [];
                } else {
                    this.selfSelectedCityIds = _.without.bind(_, this.cityIds).apply(_, this.selfSelectedCityIds)
                }
            },
            selectCity: function (city) {
                if (this.inSelectedCitys(city)) {
                    this.removeSelectedCity(city);
                } else if (this.isCountyWide(city)) {
                    this.selfSelectedCityIds = [city]
                } else if (this.countyWideSelected) {
                    return;
                } else {
                    this.selfSelectedCityIds.push(city);
                    this.selfSelectedCityIds = this.selfSelectedCityIds;
                }
                //获取已选择城市
                if (this.changeSelectedCityIds) {
                    this.changeSelectedCityIds(this.selfSelectedCityIds);
                }
            },
            inSelectedCitys: function (city) {
                if (_.indexOf(this.selfSelectedCityIds, city) !== -1) {
                    return true;
                } else {
                    return false;
                }
            },
            refineDate: function (data) {
                var _self = this;
                var DATA = {
                    useableArr: [],
                    disableArr: []
                };
                for (var j = 0; j < data.length; j++) {
                    if (_self.getDataByCityId(data[j].id)) {
                        DATA.useableArr.push(data[j]);
                    } else {
                        DATA.disableArr.push(data[j]);
                    }
                }

                return DATA;
            },
            handleUploadSuccess: function (rs) {
                let _self = this;
                if (rs.errno === 0) {
                    if (rs.data.length > 0) {
                        // 差异化有效无效城市
                        var newDate = _self.refineDate(rs.data);
                        // 有效城市加载
                        _self.selfSelectedCityIds.splice(0, _self.selfSelectedCityIds.length);
                        for (var i = 0; i < newDate.useableArr.length; i++) {
                            _self.selfSelectedCityIds.push(newDate.useableArr[i].id);
                        }

                        // 展示无效城市提示
                        var disableArr = [];
                        for (var i = 0; i < newDate.disableArr.length; i++) {
                            disableArr.push(newDate.disableArr[i].name);
                        }

                        disableArr.length && (
                                this.$message.info(disableArr.join(',') + '为无效城市'));

                        if (_self.changeSelectedCityIds) {
                            _self.changeSelectedCityIds(_self.selfSelectedCityIds);
                        }
                    } else {
                        this.$message.info('上传了' + rs.data.city_ids.length + '个城市');
                    }
                } else {
                    this.$message.error(rs.errmsg);
                }
            }
        },
        data: function () {
            return {
                citysGroup: {},
                citys: {},
                activeIndex: 0,
                cityIds: [],
                dialogVisible: false,
                selfSelectedCityIds: this.selectedCityIds ? this.selectedCityIds : []
            }
        }
    }
</script>

<style lang="less"
       rel="stylesheet/less"
       scoped>
    .bindCityBox {
        margin-top: 10px;
        margin-left: 27%;
        .selectedCityBlock {
            .selectedCityBlockTitle {
                display: inline-block;
                width: 100px;
                height: 30px;
                text-align: center;
                border: 1px solid #eeeeee;
                font-size: 14px;
                border-bottom: none;
                z-index: 1000;
                position: relative;
                top: 1px;
                background-color: #fff;
                border-top-left-radius: 5px;
                border-top-right-radius: 5px;
                letter-spacing: 1px;
            }
        }
    }

    .barWrap {
        display: flex;
        height: 36px;
        .city-demo-download {
            display: inline-block;
            border: 1px solid #bbb;
            border-radius: 8px;
            padding: 0 5px;
            text-decoration: none;
            align-self: center;
        }
        .alignSelf {
            align-self: center;
        }
    }

    .city-selector-box {
        .city-selector-tab-box {
            border-bottom: solid 1px #eee;
            height: 40px;
            margin-bottom: 15px;
            a {
                text-decoration: none;
                display: inline-block;
                padding: 0 20px;
                color: #333;
                font-size: 14px;
                line-height: 40px;
                height: 40px;
                border-bottom: solid 1px transparent;
                &.active {
                    border-color: #000;
                    color: #000;
                }
            }
        }
        .city-selector-content-box {
            display: none;
            height: 250px;
            overflow-y: auto;
            &.active {
                display: block;
            }
            .city-selector-group-box {
                position: relative;
                margin-left: 20px;
                border-left: solid 2px #eee;
                padding-bottom: 15px;
                min-height: 30px;
                &:last-child {
                    padding-bottom: 0;
                }
                & > div {
                    position: absolute;
                    left: -18px;
                    border-radius: 100%;
                    background: #eee;
                    height: 34px;
                    width: 34px;
                    line-height: 34px;
                    text-align: center;
                    float: left;
                }
                span {
                    display: inline-block;
                    margin: 7px 0 7px 40px;
                    min-width: 55px;
                    cursor: pointer;
                    input {
                        display: none;
                    }
                    i {
                        width: 10px;
                        height: 10px;
                        display: inline-block;
                        border-radius: 30%;
                        background: #eee;
                    }
                    &.disabled {
                        cursor: not-allowed;
                        color: rgba(0, 0, 0, 0.3);
                    }
                    &.checked {
                        i {
                            background: #00ff21;
                        }
                    }
                }
            }
        }
    }
</style>
