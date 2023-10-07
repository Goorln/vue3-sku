<script setup>
import { onMounted, ref } from 'vue'
import axios from 'axios'
import powerSet from './power-set'
// 商品数据
const goods = ref({})
let pathMap = {}
const getGoods = async () => {
  // 1135076  初始化就有无库存的规格
  // 1369155859933827074 更新之后有无库存项（蓝色-20cm-中国）
  const res = await axios.get('http://pcapi-xiaotuxian-front-devtest.itheima.net/goods?id=1369155859933827074')
  goods.value = res.data.result
  pathMap = getPathMap(goods.value)
  console.log(pathMap);
  initDisabledStatus(goods.value.specs, pathMap)
}
onMounted(() => getGoods())

// 切换选中状态
const changeSelectedStatus = (item, val) => {
  if (val.disabled) return
  // item:同一排对象 val: 当前点击项
  // 如果当前已经激活，就取消激活
  if (val.selected) {
    val.selected = false
  } else {
    // 如果当前未激活，就把同排其他取消激活，再把自己激活
    item.values.forEach(val => val.selected = false)
    val.selected = true
  }
  // 点击按钮时更新
  updateDisabledStatus(goods.value.specs, pathMap)
  // 产出SKU对象数据
  // 有效sku？已经选择的数组中找不到undefined,那么用户已经选择了所有有效的sku
  const index = getSelectedValues(goods.value.specs).findIndex(item => item === undefined)
  if (index > -1) {
    console.log('找到了，信息不完整');
  } else {
    // console.log('没找到，信息完整，可以产出');
    // 获取sku对象
    const key = getSelectedValues(goods.value.specs).join('-')
    const skuIds = pathMap[key]
    // console.log(pathMap, key, skuIds);
    const skuObj = goods.value.skus.find(item => item.id === skuIds[0])
    console.log('sku对象为', skuObj);
  }
}
// 生成有效路径字典对象
const getPathMap = (goods) => {
  const pathMap = {}
  // 1.根据skus字段生成有效的sku数组
  const effectiveSkus = goods.skus.filter(sku => sku.inventory > 0)

  // 2，根据有效的sku使用算法（子集算法） [1,2] => [[1],[2],[1,2]]
  effectiveSkus.forEach(sku => {
    // 2.1 获取匹配的valueName 组成的数组
    const selectedValArr = sku.specs.map(val => val.valueName)
    // 2.2 使用算法获取子集
    const valueArrPowerSet = powerSet(selectedValArr)
    // 3 把得到的子集生成最终的字典对象
    valueArrPowerSet.forEach(arr => {
      //初始化key 数组join -> 字符串 对象的key
      const key = arr.join('-')
      //如果已经存在当前key 就往数组里面直接添加skuId
      if (pathMap[key]) {
        pathMap[key].push(sku.id)
      } else {
        // 如果不存在当前key 就直接做赋值
        pathMap[key] = [sku.id]
      }
    })
  })
  return pathMap
}

// 初始化禁用状态
// 思路：遍历每一个规格的对象，使用name字段作为key去路径地点pathMap中做匹配，匹配不上则为false
const initDisabledStatus = (specs, pathMap) => {
  specs.forEach(spec => {
    spec.values.forEach(val => {
      if (pathMap[val.name]) {
        val.disabled = false
      } else {
        val.disabled = true
      }
    })
  })
}

//获取选中项的匹配数组
const getSelectedValues = (specs) => {
  const arr = []
  specs.forEach(spec => {
    // 目标： 找到values中selected为true的项，然后把他的name字段添加数组对应的位置
    const selectedVal = spec.values.find(item => item.selected === true)
    arr.push(selectedVal ? selectedVal.name : undefined)
  })


  return arr
}

// 切换时更新禁用状态
// 思路： 按照顺序得到规格选中项的数组['蓝色','20cm',undefined]
const updateDisabledStatus = (specs, pathMap) => {
  // 遍历每一个规格
  specs.forEach((spec, index) => {
    const selectedValues = getSelectedValues(specs)
    spec.values.forEach(val => {
      // 把name字段的值填充到对应的位置
      selectedValues[index] = val.name
      // 过滤掉undefined项使用join方法形成一个有效的key
      const key = selectedValues.filter(value => value).join('-')
      // 使用key去pathMap中进行匹配，匹配不上，则当前项禁用
      if (pathMap[key]) {
        val.disabled = false
      } else {
        val.disabled = true
      }
    })
  })
}

</script>

<template>
  <div class="goods-sku">
    <dl v-for="item in goods.specs" :key="item.id">
      <dt>{{ item.name }}</dt>
      <dd>
        <template v-for="val in item.values" :key="val.name">
          <!-- 图片类型规格 -->
          <img v-if="val.picture" :class="{selected:val.selected,disabled:val.disabled}" @click="$event=>changeSelectedStatus(item,val)" :src="val.picture" :title="val.name">
          <!-- 文字类型规格 -->
          <span v-else :class="{selected:val.selected,disabled:val.disabled}" @click="$event=>changeSelectedStatus(item,val)">{{ val.name }}</span>
        </template>
      </dd>
    </dl>
  </div>
</template>

<style scoped lang="scss">
@mixin sku-state-mixin {
  border: 1px solid #e4e4e4;
  margin-right: 10px;
  cursor: pointer;

  &.selected {
    border-color: #27ba9b;
  }

  &.disabled {
    opacity: 0.6;
    border-style: dashed;
    cursor: not-allowed;
  }
}

.goods-sku {
  padding-left: 10px;
  padding-top: 20px;

  dl {
    display: flex;
    padding-bottom: 20px;
    align-items: center;

    dt {
      width: 50px;
      color: #999;
    }

    dd {
      flex: 1;
      color: #666;

      > img {
        width: 50px;
        height: 50px;
        margin-bottom: 4px;
        @include sku-state-mixin;
      }

      > span {
        display: inline-block;
        height: 30px;
        line-height: 28px;
        padding: 0 20px;
        margin-bottom: 4px;
        @include sku-state-mixin;
      }
    }
  }
}
</style>