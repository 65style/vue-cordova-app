<template>
  <div class="jobinfo">
    <van-field name="zhiye" label="职业身份" required autofocus>
      <template #input>
        <van-radio-group v-model="rz.zhiye" direction="horizontal">
          <van-radio name="1">法人代表</van-radio>
          <van-radio name="2">优质精英</van-radio>
          <van-radio name="3">普通受薪</van-radio>
          <van-radio name="4">现金流</van-radio>
        </van-radio-group>
      </template>
    </van-field>

    <van-field
      v-model="rz.gongsi"
      name="gongsi"
      type="text"
      label="公司名称"
      placeholder="公司名称"
      required
      autofocus
      maxlength="20"
    />
    <van-field
      v-model="rz.gsaddress"
      name="gsaddress"
      type="text"
      label="公司地址"
      placeholder="公司地址"
      maxlength="20"
    />
    <van-field name="ifzhizhao" label="有无执照" autofocus>
      <template #input>
        <van-radio-group v-model="rz.ifzhizhao" direction="horizontal">
          <van-radio name="1">有</van-radio>
          <van-radio name="2">无</van-radio>
        </van-radio-group>
      </template>
    </van-field>

    <van-field
      v-model="rz.gufen"
      name="gufen"
      type="number"
      label="所占股份"
      placeholder="所占股份（%）"
      maxlength="3"
    />

    <van-field name="rzWorkYear" label="工作年限">
      <template #input>
        <van-radio-group v-model="rz.rzWorkYear" direction="horizontal">
          <van-radio name="3">1年以内</van-radio>
          <van-radio name="1">1～3年</van-radio>
          <van-radio name="2">3年以上</van-radio>
        </van-radio-group>
      </template>
    </van-field>

    <van-field
      v-model="rz.rzjyfw"
      name="rzjyfw"
      type="text"
      label="经营范围"
      placeholder="经营范围"
      maxlength="10"
    />
    <van-field
      readonly
      clickable
      autofocus
      name="zhizhao"
      :value="rz.zhizhao"
      label="注册时间"
      placeholder="点击选择注册时间"
      @click="showPickerDate = true"
    />
    <van-popup v-model="showPickerDate" position="bottom">
      <van-datetime-picker
        v-model="currentDate"
        type="date"
        title="选择年月日"
        :min-date="minDate"
        :max-date="maxDate"
        @cancel="showPickerDate = false"
        @confirm="onConfirmDate"
      />
    </van-popup>

    <van-field name="forbid" label="是否禁入行业">
      <template #input>
        <van-radio-group v-model="rz.forbid" direction="horizontal">
          <van-radio name="1">是</van-radio>
          <van-radio name="2">否</van-radio>
        </van-radio-group>
      </template>
    </van-field>

    <van-field
      v-model="rz.yingyee"
      name="yingyee"
      type="number"
      label="年营业额"
      placeholder="年营业额（万元/年）"
      maxlength="10"
    />
    <van-field
      v-model="rz.yuejing"
      name="yuejing"
      type="number"
      label="月净收入"
      placeholder="月净收入（元/月）"
      maxlength="10"
    />
    <van-field name="xugua" label="需挂靠单位" required autofocus>
      <template #input>
        <van-radio-group v-model="rz.xugua" direction="horizontal">
          <van-radio name="1">是</van-radio>
          <van-radio name="2">否</van-radio>
        </van-radio-group>
      </template>
    </van-field>
    <van-field name="xiahu" label="单位能否下户" required autofocus>
      <template #input>
        <van-radio-group v-model="rz.xiahu" direction="horizontal">
          <van-radio name="1">是</van-radio>
          <van-radio name="2">否</van-radio>
        </van-radio-group>
      </template>
    </van-field>
    <van-field name="zuoji" label="单位座机" required autofocus>
      <template #input>
        <van-radio-group v-model="rz.zuoji" direction="horizontal">
          <van-radio name="1">需安装</van-radio>
          <van-radio name="2">有座机</van-radio>
        </van-radio-group>
      </template>
    </van-field>
    <van-field name="fuye" label="有无副业">
      <template #input>
        <van-radio-group v-model="rz.fuye" direction="horizontal">
          <van-radio name="1">有</van-radio>
          <van-radio name="2">无</van-radio>
        </van-radio-group>
      </template>
    </van-field>
    <van-field
      v-model="rz.fyshouru"
      name="fyshouru"
      type="number"
      label="副业收入"
      placeholder="副业收入（元/月）"
      maxlength="10"
    />
  </div>
</template>

<script>
export default {
  props: {
    rz: {
      type: Object,
      required: true,
    },
  },
  data() {
    return {
      showPickerDate: false,
      minDate: new Date(
        new Date().getFullYear() - 10,
        new Date().getMonth(),
        new Date().getDate(),
      ),
      maxDate: new Date(),
      currentDate: new Date(),
    }
  },
  mounted() {
    this.currentDate = this.rz.zhizhao || this.currentDate
  },
  methods: {
    // 日期选择器确认 注册时间
    onConfirmDate(date) {
      let year = date.getFullYear()
      let month = date.getMonth() + 1
      month = month < 10 ? '0' + month : month
      let day = date.getDate()
      day = day < 10 ? '0' + day : day
      this.rz.zhizhao = `${year}-${month}-${day}`
      this.showPickerDate = false
    },
  },
}
</script>

<style></style>
