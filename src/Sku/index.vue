<script setup>
import { onMounted, ref } from 'vue'
import axios from 'axios'
import powerSet from './power-set'
// 商品数据
const goods = ref({})
const getGoods = async () => {
  // 1135076  初始化就有无库存的规格
  // 1369155859933827074 更新之后有无库存项（蓝色-20cm-中国）
  const res = await axios.get('http://pcapi-xiaotuxian-front-devtest.itheima.net/goods?id=1369155859933827074')
  goods.value = res.data.result
  const pathMap1 = getPathMap(goods.value)
  console.log(pathMap1);
}
onMounted(() => getGoods())

// 切换选中状态
const changeSelectedStatus = (item, val) => {
  // item:同一排对象 val: 当前点击项
  // 如果当前已经激活，就取消激活
  if (val.selected) {
    val.selected = false
  } else {
    // 如果当前未激活，就把同排其他取消激活，再把自己激活
    item.values.forEach(val => val.selected = false)
    val.selected = true
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

</script>

<template>
  <div class="goods-sku">
    <dl v-for="item in goods.specs" :key="item.id">
      <dt>{{ item.name }}</dt>
      <dd>
        <template v-for="val in item.values" :key="val.name">
          <!-- 图片类型规格 -->
          <img v-if="val.picture" :class="{selected:val.selected}" @click="$event=>changeSelectedStatus(item,val)" :src="val.picture" :title="val.name">
          <!-- 文字类型规格 -->
          <span v-else :class="{selected:val.selected}" @click="$event=>changeSelectedStatus(item,val)">{{ val.name }}</span>
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