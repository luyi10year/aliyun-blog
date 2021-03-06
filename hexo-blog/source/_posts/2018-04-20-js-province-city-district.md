title: '一颗关于省市区的树'
date: 2018-04-20 14:17:54
tags:
    - vue
    - js
---
如何绘制一个省市区三级可选择的树？
<!--more-->


### 开始

首先你拥有的数据结构 所有省市区的信息列表 以及已经选中的信息
用的是element-ui的 el-tree
```javascript
const cityStorage = {
    provinceList:[
        {id: 1, provinceId: "110000", name: "北京市"}
    ],//所有省
    cityList:[
        {id: 1, cityId: "110100", name: "北京市", provinceId: "110000", zipCode: "102600"}
    ],//所有市
    districtList:[
        {id: 1, districtId: "110101", name: "东城区", cityId: "110100"}
    ],//所有区
    }
const selectList = [
    { provinceId: "110000",cityId: "110100",districtId: "110101"}
] // 所有选中的省市区 ID 保存的时候也是这个格式
```

### 按需渲染
首先 作为有相对要求的开发人员 不会考虑说 直接的去渲染出整个树 那整个省市区加载的速度绝对会是感人的

那么 可行的解决方法是 一开始 只展示 所有省的信息 点击展开 时再去渲染下一层 数据

这个对应关系 相对还容易找 每次点击展开能获得当前的层级和id 根据层级和id去对应的city和district中过滤就行

这里分享一个小技巧 不通过判断的方式去对应 而是通过数据的方式
```javascript
//level , id
const levelConfig = {
     1: {
        idLabel: 'cityId',
        fetchLabel: 'cityList',
        perIdLabel: 'provinceId'
     },
     2: {
        idLabel: 'districtId',
        fetchLabel: 'districtList',
        perIdLabel: 'cityId'
     }
}

// 那么过滤就可以这么写
cityStorage[levelConfig[level].fetchLabel].filter(item => item[levelConfig[level].perIdLabel] == id)
```
获取数据 然后加载对应下一层 一切到现在为止 都还可以
![new-tree](/assets/blogImg/tree/new-tree.png)


### 赋值渲染
再往下 如果我有初始数据呢？

在只展示省信息的情况下 结合前面给的数据格式 怎么展示 这个省是 全选 半选(表示省中有选择的市或者区但没选全) 和 不选 ？

第一 你需要设法知道省份满足全选的条件
第二 你需要设法知道已经选择的情况

所以这个时候 需要做的 是计数 也就 ++
遍历一遍 cityStorage.provinceList 和 cityStorage.cityList
往Map中初始化 provinceId cityId 对应的计数
在遍历 cityStorage.districtList 过程中往Map 对应provinceId cityId 增加计数

那么 有没有什么别的基础数据 是要在这个时候初始化的呢？
例如 只给你一个 districtId 你怎么才能最快的 找到他对应的 cityId 和 provinceId
或者 只给你一个 id 怎么最快找到他 对应的 name 呢
Map
我们可以构建一个Map来记录我们需要的信息
districtId:cityId
cityId:provinceId
那么 我可以通过 Map[districtId] 找到cityId Map[Map[districtId]] 找到 provinceId
id 和 name 的对应关系 也是如此
而这些 可以在 计数的 过程进行

接着 通过已经选择地区 的 列表 获取provinceId cityId的数据 的计数

两份数据都有了 

在渲染 省的时候 判断 两份Map中对应的计数 是否相同来渲染勾选

那半选的状态怎么表示呢？el-tree并不支持设置半选的状态，必须是通过数据的形式呢？
![edit-tree](/assets/blogImg/tree/edit-tree.png)

通过模拟子节点的方法 当满足不全选的情况 模拟两个子节点
```javascript
var children = [
    {
      id:provinceId+'111',
      label:name,
      type:'none'
    },
    {
      id:provinceId+'222',
      label:name,
      type:'none'
    }
]
```
然后选中 其中一个 父元素自然就是半选状态

### 保存提交
最后是保存提交时候的数据处理

由于模拟了半选状态 所以最后获取到的选中的数据 会有两种

一种常规的6位 还是一种是模拟的d{6}xxx

而且如果 是出现这种d{6}xxx的数据 代表的是它所在的一级有些被选中了 而这些数据还没有出现在 渲染树中

这是就 需要有一个数据结构记录 这种情况

在已选择的数据 初始化计数的时候 新构建一个Map 存储 provinceId cityId出现的数据的下标(我这边保存的是districtId)
provinceId:[districtId,districtId,districtId]
cityId:[districtId,districtId,districtId]
至此 我们最后能拿到的 选中的 id 有 [310000,410000111,510100,610101]
此时这份数据中 有provinceId cityId districtId 以及 模拟的半选数据 怎么尽可能的优雅的生成我们需要的格式呢？

首先是分类 可以发现 xx0000表示的是省  xxx000 xxxx00 表示的是市
```javascript
let _zeo = item.match(/(0+)$/g),
           type = _zeo ? _zeo[0] : '0'
          switch (type) {
            case '0':
              districtList.push(item)
              break;
            case '00':
            case '000':
              cityList.push(item)
              break;
            case '0000':
              provinceList.push(item)
              break;
          }
```
而这种410000111 数据 可以通过 先前的 Map 将数据并入 districtList 中

接着就是净化数据

省选择了 不需要市的所有id 市选择了 不需要区的所有id

总结 判断条件
```javascript
var sub = item.substr(0,2)
var re =new RegExp('\^' +sub+'\\d{4}'); // 省
// var sub = item.substr(0,4)
// var re =new RegExp('\^' +sub+'\\d{2}'); // 市
cityList.filter(code=>!code.match(re))
```
最后 provinceList cityList districtList 都是有效的选中值
遍历一遍 cityStorage.districtList 将其中在provinceList cityList中存在id的数据并入 districtList中

此时 districtList 是最终有效的所有选中的 districtId值
此时 cityId = Map[districtId] province=Map[cityId]



### 展示归并

保存后希望展示成
当选择了一个省份全部地区的时候展示省份名称
当选择了一个省份下的部分市时展示市的名称
当选择了一个市下的所有地区时,只展示市的名称
当选择了一个市下的部分地区时,括号内展示地区名称
![edit-tree](/assets/blogImg/tree/name.png)

首先还是通过计数 获取 已经选中的有效的 provinceList cityList districtList
数据格式{provinceId: "110000",cityId: "110100",districtId: "110101"}

构建一个存 selectNameList 用于存放已经选中的 name name可以通过前面的Map[id]获取

provinceList 选择的省没问题 直接推入

cityList 遍历 构建新的数据 格式 {provinceId: "110000",cityId: "110100",districtId: "1"}
并入 districtList 中

对districtList 根据 cityId 排序

最后 遍历 districtList 通过标记判断每次是否是重复的cityId  设置数组 indeterminateNameList 记录不是全选的市的name
不重复 将标记记为当前cityId 如果上一个元素的 districtCode 是 1 将 indeterminateNameList 存入 selectNameList 中 
selectNameList.push('(<span class="c-dis">' + indeterminateNameList.join('，') + '</span>)')
districtId == 1 全选 存入 selectNameList
districtId != 1 不是全选 存入 indeterminateNameList = [cityId]
在这个过程中 有需要 还可以记录 cityId 和 indeterminateNameList的管理关系

最终 selectNameList.join('，') 


### 源码
```html
<template>
    <div>
        <div class="title">
            <span>管理配送模板</span>
        </div>
        <p class="btn-wrap">
            <el-button type="primary" @click="showTplLayer()">新增配送模板</el-button>
        </p>
        <div class="section">
            <div v-if="isReady">
                                <div class="section-item" v-for="(item,index) in distributionList" :key="index">
                                    <div class="block-item o-h">
                                        <span>模板名称：{{item.name}}</span>
                                        <el-button class="fl-r" type="primary" size="small" @click="deleteTpl(index)">删除</el-button>
                                        <el-button class="fl-r" style="margin-right: 10px" type="primary" size="small"
                                                   @click="showTplLayer(index+'')">修改
                                        </el-button>
            
                                    </div>
                                    <div class="block-item">
                                        <span v-if="item.type==1">限制类型：仅发以下地区</span>
                                        <span v-else-if="item.type==2">限制类型：以下地区不发货</span>
                                        <span v-else-if="item.type==3">限制类型：限制仅发货省份中的不发货区县</span>
                                    </div>
                                    <div class="block-item">
                          <span>限制地区：<span v-if="isCityDetail" v-html="limitName[index]"></span>
                          <i v-else class="el-icon-loading c-red"></i>
                          </span>
                                    </div>
                                    <div class="block-item">
                                        <span>模板描述：{{item.desc}}</span>
                                    </div>
                                </div>
                            </div>
            <div v-else class="block-item-loading">
                 <i class="el-icon-loading c-red"></i>
            </div>


        </div>
        <el-dialog :title="tplLayer.title" size="large" v-model="tplLayer.show">
            <div class="price-ct">
                <div class="input-ct">
                    <div>
                        <span>模板名称：</span>
                        <el-input v-model="tplLayer.inputName" placeholder="请输入内容"></el-input>
                    </div>
                    <div>
                        <span>限制类型：</span>
                        <el-radio-group v-model="tplLayer.radioType">
                            <el-radio :label="1">仅发以下地区</el-radio>
                            <el-radio :label="2">以下地区不发货</el-radio>
                        </el-radio-group>
                    </div>
                    <div>
                        <span>限制地区：</span>
                        <div class="limit-tree-wrap">
                            <!--:load="loadLimit"-->
                            <el-tree
                                    :data="limitData"
                                    show-checkbox
                                    node-key="id"
                                    ref="tree"
                                    @node-expand="nodeExpand"
                                    @node-collapse="nodeCollapse"
                                    @current-change="checkChange"
                                    @node-click="nodeClick"
                                    highlight-current
                                    :props="defaultProps">
                            </el-tree>
                        </div>
                    </div>
                    <div>
                        <span>模板描述：</span>
                        <el-input v-model="tplLayer.inputDesc" placeholder="请输入内容"></el-input>
                    </div>
                </div>
            </div>
            <div slot="footer" class="dialog-footer">
                <el-button type="primary" @click="saveTpl">确 定</el-button>
                <el-button @click="cancelTpl">取 消</el-button>
            </div>
        </el-dialog>
        <el-dialog title="提示" v-model="dialogVisible" size="tiny">
            <span>确定删除【{{distributionListName}}】模板？</span>
            <span slot="footer" class="dialog-footer">
                <el-button @click="dialogVisible = false">取 消</el-button>
                <el-button type="primary" @click="confirmDelete">确 定</el-button>
            </span>
        </el-dialog>
    </div>
</template>
<script>
    import {mapState, mapActions} from 'vuex'
    export default {
        name: 'distributionTemplate',
        created (){
            this.initView()
        },
        data: () => ({
        isReady:false, 
        isCityDetail:false,
        tplLayer: {
            show: false,
            title: '',
            inputName: '',
            inputDesc: '',
            radioType: 1,
            index: 0,
            cityArr: [],
        },
        limitName:[],
        localAllCity: {
            provinceList: [],
            cityList: [],
            districtList: []
        },
        dialogVisible: false,
        index: null,
        distributionListName: '',
        distributionList: [],
        limitData: [
//              {
//                id: 1,
//                label: '一级 1',
//                children: [
//                    {
//                      id: 4,
//                      label: '二级 1-1',
//                      children: [
//                          {
//                          id: 9,
//                          label: '三级 1-1-1'
//                          }
//                      ]
//                    }]
//              }
        ],
        defaultProps: {
            children: 'children',
            label: 'label'
        },
        mapLimitData:{

        },
        limitCheckList:{
            province:[],
            city:[],
            district:[]
        },
        mapAllAreaName:{},
        mapAllArea:{},
        mapAllAreaList:[],
        mapAreaLength:{

        },
        mapProvinceAreaLength:{
//          1:1
        },
        mapCityAreaLength:{
//          1:{
//              _len:1,
//              id:''
//          }
        },
        checkMapAreaLength:{},
        indeterminateArea:{}
    }),
    computed: {
    ...mapState(['allCity'])
    },
    methods: {
    ...mapActions(['getDistributionTemplate', 'deleteDeliverTpl', 'saveDeliverTpl', 'getAllCity', 'callSetNotice']),
            initView() {
            if (!Util.storage.get('CityStorage')) {
                this.getAllCity()
                    .then(() => {
                    Util.storage.set('CityStorage', this.allCity.data)
                this.initList()
            })
            } else {
                this.initList()
            }

        },
        initList() {
            this.setAreaBase()
            this.callGetDistributionTemplate()
        },
        setAreaBase(){
            var allList = Util.storage.get('CityStorage')
            this.localAllCity.provinceList = allList.provinceList
            this.localAllCity.cityList = allList.cityList
            this.localAllCity.districtList = allList.districtList

            //初始化处理省 市 区 全选计数
            this.localAllCity.provinceList.forEach((item)=>{
                this.mapProvinceAreaLength[item.provinceId] = 0
            this.mapAllAreaName[item.provinceId] = item.name
//          this.mapAllAreaList[item.provinceId] = []
        })
            this.localAllCity.cityList.forEach((item)=>{
                this.mapCityAreaLength[item.cityId] = {
                len:0,
                id:item.provinceId
            }
            this.mapAllArea[item.cityId] = item.provinceId
//          this.mapAllAreaList[item.cityId] = []
//          this.mapAllAreaList[item.provinceId].push(item.cityId)
            this.mapAllAreaName[item.cityId] = item.name
        })

            this.localAllCity.districtList.map((item)=>{
                if(this.mapCityAreaLength[item.cityId]){
                this.mapCityAreaLength[item.cityId].len++
                this.mapProvinceAreaLength[this.mapCityAreaLength[item.cityId].id] ++
            }
            item.provinceId = this.mapAllArea[item.cityId]
            this.mapAllArea[item.districtId] = item.cityId
//          this.mapAllAreaList[item.cityId].push(item.districtId)
            this.mapAllAreaName[item.districtId] = item.name
        })
        },
        callGetDistributionTemplate(){
            this.isReady = false
            this.getDistributionTemplate()
                .then((rs) => {
                //遍历城市编码转换城市名称
                this.distributionList = rs
            // 延时加载
//            this.$nextTick(() => {
//              this.setLimitAreaName()
//            })
            this.isReady = true
            this.isCityDetail = false
            this.limitName = []
            setTimeout(function(){
                this.setLimitAreaName()
                this.isCityDetail = true
            }.bind(this),1000)

        })
        },
        setLimitAreaName(){
            for (let i = 0, j = this.distributionList.length; i < j; i++) {
                let list = this.distributionList[i].limitAreaList
                let _checkMapAreaLength = {}, _province = [], _city = [], _district = []
                for (let n = 0, m = list.length; n < m; n++) {
                    let item = list[n]
                    if (!_checkMapAreaLength[item.provinceCode]) {
                        _checkMapAreaLength[item.provinceCode] = 1
                    } else {
                        _checkMapAreaLength[item.provinceCode]++
                    }
                    if (!_checkMapAreaLength[item.cityCode]) {
                        _checkMapAreaLength[item.cityCode] = 1
                    } else {
                        _checkMapAreaLength[item.cityCode]++
                    }
                }
                for (let i in _checkMapAreaLength) {
                    // 选择的 省
                    if (_checkMapAreaLength[i] == this.mapProvinceAreaLength[i]) {
                        _province.push(i)
                    }
                    // 选择 的 市
                    if (this.mapCityAreaLength[i] && _checkMapAreaLength[i] == this.mapCityAreaLength[i].len) {
                        _city.push(i)
                    }
                }
                _district = list.filter(item => !_province.includes(item.provinceCode) && !_city.includes(item.cityCode))
                let _sNameList = [], _limitList = []
                _limitList = _limitList.concat(_district)
                _province.forEach((item) => {
                    _sNameList.push(this.mapAllAreaName[item])
            })
                _city.forEach((item) => {
                    if (!_province.includes(this.mapAllArea[item])) {
                    let _opts = {
                        provinceCode: '',
                        cityCode: item,
                        districtCode: 1,
                    }
                    _limitList.push(_opts)
                }

            })
                _limitList.sort((a, b) => {
                    return a.cityCode - b.cityCode
                })
                let current = '', _subList = []
                _limitList.forEach((item, idx) => {
                    if (item.cityCode != current) {
                    current = item.cityCode
                    if (_limitList[idx - 1] && _limitList[idx - 1].districtCode != 1) {
                        _sNameList.push(this.mapAllAreaName[_limitList[idx - 1].cityCode] + '(<span style="color:#aaa">' + _subList.join('，') + '</span>)')
                    }
                    if (item.districtCode == 1) {
                        _sNameList.push(this.mapAllAreaName[item.cityCode])
                    } else {
                        _subList = [this.mapAllAreaName[item.districtCode]]
                    }
                }
            else {
                    _subList.push(this.mapAllAreaName[item.districtCode])
                }

            })
                this.limitName.push(_sNameList.join('，'))
            }
        },
        showTplLayer(index){
            this.index = index || null
            this.tplLayer.title = index ? '修改配送模板' : '新增配送模板'
            this.tplLayer.show = true
            if (index) {
                var _thisTplObj = this.distributionList[index]
                this.tplLayer.inputName = _thisTplObj.name
                this.tplLayer.inputDesc = _thisTplObj.desc
                this.tplLayer.radioType = _thisTplObj.type
                this.setLimitCheckList(_thisTplObj.limitAreaList)
            }
            else {
                this.tplLayer.inputName = ''
                this.tplLayer.inputDesc = ''
                this.tplLayer.radioType = 1
            }
            this.limitDefault()
        },
        limitDefault(){
            let _check = []
            this.localAllCity.provinceList.forEach((item) => {
                let _opts = {
                    id: item.provinceId,
                    label: item.name,
                    type:'provinceId',
                    provinceId:item.provinceId,
                    cityId:'',
                    districtId:'',
                    children:[
                        {
                            id:item.provinceId+'111',
                            label:item.name,
                            type:'none'
                        },
                        {
                            id:item.provinceId+'222',
                            label:item.name,
                            type:'none'
                        }
                    ],
                }
                this.limitData.push(_opts)

            if(this.limitCheckList.province.includes(item.provinceId)){
                _check.push(item.provinceId)
                _check.push(item.provinceId+'111')
                _check.push(item.provinceId+'222')
            } else if(this.checkMapAreaLength[item.provinceId]){
                _check.push(item.provinceId+'111')
            }
        })
            this.setCheckLimitArea(_check)
        },
        mockData(){
            let _list = [
                {
                    provinceCode:'110000',
                    cityCode:'110100',
                    districtCode:'110228'
                },
                {
                    provinceCode:'110000',
                    cityCode:'110100',
                    districtCode:'110229'
                },
//          {
//            provinceCode:'110000',
//            cityCode:'110100',
//            districtCode:'110230'
//          }
            ]
            var _len = new Array(17).fill(1)
            _len.map((item,idx)=>{
                let _code = '1101' + (idx < 9 ? '0' : '') + (idx+1)
                if(idx != 9){
                let _opts = {
                    provinceCode:'110000',
                    cityCode:'110100',
                    districtCode:_code
                }
                _list.push(_opts)
            }

        })
            return _list
        },
        setLimitCheckList(rs){
//        let list = this.mockData()
            let list = rs
            list.forEach((item) => {
                //已选择 的 省份 计数
                if (!this.checkMapAreaLength[item.provinceCode]) {
                this.checkMapAreaLength[item.provinceCode] = 1
                this.indeterminateArea[item.provinceCode] = [item.cityCode]
            } else {
                this.checkMapAreaLength[item.provinceCode]++
                if(!this.indeterminateArea[item.provinceCode].includes(item.cityCode)){
                    this.indeterminateArea[item.provinceCode].push(item.cityCode)
                }
            }
            //已选择 的 城市 计数
            if (!this.checkMapAreaLength[item.cityCode]) {
                this.checkMapAreaLength[item.cityCode] = 1
                this.indeterminateArea[item.cityCode] = [item.districtCode]
            } else {
                this.checkMapAreaLength[item.cityCode]++
                if(!this.indeterminateArea[item.cityCode].includes(item.districtCode)){
                    this.indeterminateArea[item.cityCode].push(item.districtCode)
                }
            }
            //选择的 区
            this.limitCheckList.district.push(item.districtCode)
        })
            for (let i in this.checkMapAreaLength) {
                // 选择的 省
                if (this.checkMapAreaLength[i] == this.mapProvinceAreaLength[i]) {
                    this.limitCheckList.province.push(i)
                }
                // 选择 的 市
                if (this.mapCityAreaLength[i] && this.checkMapAreaLength[i] == this.mapCityAreaLength[i].len) {
                    this.limitCheckList.city.push(i)
                }
//          if(this.mapProvinceAreaLength[i]){
//            console.log(i,this.checkMapAreaLength[i],this.mapProvinceAreaLength[i])
//          }
//          if(this.mapCityAreaLength[i]){
//            console.log(i,this.checkMapAreaLength[i],this.mapCityAreaLength[i].len)
//          }

            }

//        console.log(this.limitCheckList.city)
        },
        nodeExpand(data, node){
            let id = data.id,
                _level = node.level,
                _checked = node.checked,
                _indeterminate = node.indeterminate
            // 已经展开 back
            if (this.mapLimitData[id]) return
            this.mapLimitData[id] = true
            // 市 区 对应map
            let _levelMap = {
                1: {
                    idLabel: 'cityId',
                    fetchLabel: 'cityList',
                    perIdLabel: 'provinceId'
                },
                2: {
                    idLabel: 'districtId',
                    fetchLabel: 'districtList',
                    perIdLabel: 'cityId'
                }
            }
            let configLevel = _levelMap[_level]
            let _list = [], selectId = []
            this.localAllCity[configLevel.fetchLabel].filter(item => item[configLevel.perIdLabel] == id).forEach((item) => {
                let _opts = {
                    id: item[configLevel.idLabel]+'',
                    label: item.name,
                    provinceId: item.provinceId || '',
                    cityId: item.cityId,
                    districtId: item.districtId || '',
                    type: configLevel.idLabel
                }
                if (_level < 2) {
                _opts.children = [
                    {
                        id: item[configLevel.idLabel] + '111',
                        label: item.name,
                        type: 'none'
                    },
                    {
                        id: item[configLevel.idLabel] + '222',
                        label: item.name,
                        type: 'none'
                    }
                ]
            }
            // 父 选择
            if (_checked) {
                selectId.push(item[configLevel.idLabel])
            }
            // 父 未选择 但是有子元素选择
            else if (_indeterminate) {
                if (this.limitCheckList.city.includes(item[configLevel.idLabel])) {
                    selectId.push(item[configLevel.idLabel])
                    selectId.push(item[configLevel.idLabel]+'111')
                    selectId.push(item[configLevel.idLabel]+'222')
                } else if (this.checkMapAreaLength[item[configLevel.idLabel]]) {
                    selectId.push(item[configLevel.idLabel] + '111')
                }
                if (this.limitCheckList.district.includes(item[configLevel.idLabel])) {
                    selectId.push(item[configLevel.idLabel])
                }
            }
            _list.push(_opts)
        })
//        console.log(selectId)
            data.children = _list
            this.setCheckLimitArea(selectId)
        },
        nodeCollapse(data, node){
        },
        checkChange(data){

        },
        nodeClick(data,node){},
        setCheckLimitArea(list){
            if (!list.length) return;
            this.$nextTick(() => {
                let _checkList = this.$refs.tree.getCheckedKeys().concat(list)
                this.$refs.tree.setCheckedKeys(_checkList);
        })
        },
        loadLimit(node,resolve){
            if (node.level === 0) {
                return resolve(this.limitData);
            }
            let configLevel = {
                    idLabel:'',
                    fetchLabel:'',
                    perIdLabel:''
                },
                id = node.data.id
            if (node.level === 1) {
                configLevel.fetchLabel = 'cityList'
                configLevel.idLabel = 'cityId'
                configLevel.perIdLabel = 'provinceId'
            } else if (node.level === 2) {
                configLevel.fetchLabel = 'districtList'
                configLevel.idLabel = 'districtId'
                configLevel.perIdLabel = 'cityId'
            } else {
                return resolve([]);
            }



            let _limitData = []
            this.localAllCity[configLevel.fetchLabel].filter(item=>item[configLevel.perIdLabel] == id).forEach((item)=>{
                let _opts = {
                    id: item[configLevel.idLabel],
                    label: item.name,
                }
                _limitData.push(_opts)
        })
            return resolve(_limitData);

        },
        cancelTpl(){
            this.tplLayer.show = false
            this.limitData = []
            this.mapLimitData = {}
            this.limitCheckList = {
                province:[],
                city:[],
                district:[]
            }
            this.checkMapAreaLength = {}
            this.indeterminateArea = {}
        },
        deleteTpl(index){
            this.index = index
            this.distributionListName = this.distributionList[index].name
            this.dialogVisible = true
        },
        saveTpl(){
            var limitAreaList = []
            if (!this.tplLayer.inputName) {
                this.getNotice('请输入运费模板名称')
                return false
            }
            if (!this.tplLayer.inputDesc) {
                this.getNotice('请输入运费模板描述')
                return false
            }
            limitAreaList = this.changeSelectCodeToList(this.$refs.tree.getCheckedKeys())

//        return false

            const opts = {
                id: this.index === null ? '' : this.distributionList[this.index].id,
                name: this.tplLayer.inputName,
                desc: this.tplLayer.inputDesc,
                type: this.tplLayer.radioType,
                limitAreaList: limitAreaList,
                exceptAreaList: []
            }


            this.saveDeliverTpl(opts)
                .then(() => {
                this.callGetDistributionTemplate()
            this.cancelTpl()
        })
        },
        changeSelectCodeToList(list){
//           //xx0000 省份
//          // xxxx00 市
//          // xx0000xxx 省份半选
//          // xxxx00xxx 市半选
            let indeterminate = [],
                districtList = [],
                cityList = [],
                provinceList = []
            for (let i = 0, j = list.length; i < j; i++) {
                let item = list[i]
                if (item.length > 6) {
                    let _item = item.replace(/\d{3}$/, '')
                    if(_item.match(/0000$/g)){
                        indeterminate = indeterminate.concat(this.indeterminateArea[_item])
                    } else {
                        indeterminate.push(_item)
                    }
                    continue;
                }
                let _zeo = item.match(/(0+)$/g)
                let type = _zeo ? _zeo[0] : '0'
                switch (type) {
                    case '0':
                        districtList.push(item)
                        break;
                    case '00':
                    case '000':
                        cityList.push(item)
                        break;
                    case '0000':
                        provinceList.push(item)
                        break;
                }
            }
//        console.log('i',indeterminate)
//        console.log('d',districtList)
//        console.log('c',cityList)
//        console.log('p',provinceList)
//        return false
            indeterminate.forEach((item)=>{
                if(this.indeterminateArea[item]){
                districtList = districtList.concat(this.indeterminateArea[item])
            }
        })
            provinceList.forEach((item)=>{
                let _sub = item.substr(0,2)
                var re =new RegExp('\^' +_sub+'\\d{4}','g');
            cityList = cityList.filter((code)=>{
                    return !code.match(re)
                })
            districtList = districtList.filter((code)=>{
                    return !code.match(re)
                })
        })
            cityList.forEach((item)=>{
                let _sub = item.substr(0,4)
                var re =new RegExp('\^' +_sub+'\\d{2}','g');
            districtList = districtList.filter((code)=>{
                    return !code.match(re)
                })
        })
            let _limitList = []

            districtList.forEach((item)=>{
                let _opts = {
                    provinceCode:this.mapAllArea[this.mapAllArea[item]],
                    cityCode:this.mapAllArea[item],
                    districtCode:item
                }
                _limitList.push(_opts)
        })

            this.localAllCity.districtList.forEach((item)=>{
                if(cityList.includes(item.cityId) || provinceList.includes(item.provinceId)){
                let _opts = {
                    provinceCode:item.provinceId,
                    cityCode:item.cityId,
                    districtCode:item.districtId
                }
                _limitList.push(_opts)
            }
        })

//        console.log('d',districtList)
//        console.log('c',cityList)
//        console.log('p',provinceList)
//
//        console.log('limit',_limitList)
            return _limitList

        },
        confirmDelete(){
            this.deleteDeliverTpl({
                templateId: this.distributionList[this.index].id
            }).then(() => {
                this.callGetDistributionTemplate()
            this.dialogVisible = false
        })
        },
        getNotice(msg){
            const notice = {
                isShow: true,
                msg: msg,
            }
            this.callSetNotice(notice)
        },
    }
    }
</script>
<style lang='less' rel="stylesheet/less" scoped>
    .block-item-loading{
        text-align: center;
        line-height: 400px;
        font-size: 30px;
    }
    .slide-fade-enter, .slide-fade-leave-to {
        transform: translateX(60px);
        opacity: 1;
    }

</style>

```
























