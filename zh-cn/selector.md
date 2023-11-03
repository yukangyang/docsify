> userSelector.vue   
 
 ![userSelector](../image/userSelector.png)

```html
<template>
  <a-modal v-model:visible="visible" title="用户选择" :width="1000" :mask-closable="false" :destroy-on-close="true" @ok="handleOk" @cancel="handleClose">
    <a-row :gutter="10">
      <a-col :span="7">
        <a-card size="small" :loading="cardLoading" class="selectorTreeDiv">
          <a-tree v-if="treeData" v-model:expandedKeys="defaultExpandedKeys" :tree-data="treeData" :field-names="treeFieldNames" @select="treeSelect"> </a-tree>
        </a-card>
      </a-col>
      <a-col :span="11">
        <div class="table-operator" style="margin-bottom: 10px">
          <a-form ref="searchFormRef" name="advanced_search" class="ant-advanced-search-form" :model="searchFormState">
            <a-row :gutter="24">
              <a-col :span="12">
                <a-form-item name="searchKey">
                  <a-input v-model:value="searchFormState.searchKey" placeholder="请输入用户名" />
                </a-form-item>
              </a-col>
              <a-col :span="12">
                <a-button type="primary" class="primarySele" @click="loadData()"> 查询 </a-button>
                <a-button class="snowy-buttom-left" @click="reset()"> 重置 </a-button>
              </a-col>
            </a-row>
          </a-form>
        </div>
        <div class="user-table">
          <a-table
            ref="table"
            size="small"
            :columns="commons"
            :data-source="tableData"
            :expand-row-by-click="true"
            :loading="pageLoading"
            bordered
            :pagination="false"
          >
            <template #title>
              <span>待选择列表 {{ tableRecordNum }} 条</span>
              <div v-if="!radioModel" style="float: right">
                <a-button type="dashed" size="small" @click="addAllPageRecord">添加当前数据</a-button>
              </div>
            </template>
            <template #bodyCell="{ column, record }">
              <template v-if="column.dataIndex === 'action'">
                <a-button type="dashed" size="small" @click="addRecord(record)">添加</a-button>
              </template>
              <template v-if="column.dataIndex === 'category'"> {{ $TOOL.dictTypeData('ROLE_CATEGORY', record.category) }} </template>
            </template>
          </a-table>
          <div class="mt-2">
            <a-pagination
              v-if="!isEmpty(tableData)"
              v-model:current="current"
              v-model:page-size="pageSize"
              :total="total"
              size="small"
              showSizeChanger
              @change="paginationChange"
            />
          </div>
        </div>
      </a-col>
      <a-col :span="6">
        <div class="user-table">
          <a-table
            ref="selectedTable"
            size="small"
            :columns="selectedCommons"
            :data-source="selectedData"
            :expand-row-by-click="true"
            :loading="selectedTableListLoading"
            bordered
          >
            <template #title>
              <span>已选择: {{ selectedData.length }}</span>
              <div v-if="!radioModel" style="float: right">
                <a-button type="dashed" danger size="small" @click="delAllRecord">全部移除</a-button>
              </div>
            </template>
            <template #bodyCell="{ column, record }">
              <template v-if="column.dataIndex === 'action'">
                <a-button type="dashed" danger size="small" @click="delRecord(record)">移除</a-button>
              </template>
            </template>
          </a-table>
        </div>
      </a-col>
    </a-row>
  </a-modal>
</template>
```

```js
<script setup name="userSelectorPlus">
  import { message } from "ant-design-vue";
  import { remove, isEmpty } from "lodash-es";
  // 弹窗是否打开
  const visible = ref(false);
  // 主表格common
  const commons = [
    {
      title: "操作",
      dataIndex: "action",
      align: "center",
      width: 80
    },
    {
      title: "用户名",
      dataIndex: "name",
      ellipsis: true
    },
    {
      title: "账号",
      dataIndex: "account"
    }
  ];
  // 选中表格的表格common
  const selectedCommons = [
    {
      title: "操作",
      dataIndex: "action",
      align: "center",
      width: 80
    },
    {
      title: "用户名",
      dataIndex: "name",
      ellipsis: true
    }
  ];
  // 主表格的ref 名称
  const table = ref();
  // 选中表格的ref 名称
  const selectedTable = ref();
  const tableRecordNum = ref();
  const searchFormState = ref({});
  const searchFormRef = ref();
  const cardLoading = ref(true);
  const pageLoading = ref(false);
  const selectedTableListLoading = ref(false);
  // 替换treeNode 中 title,key,children
  const treeFieldNames = { children: "children", title: "name", key: "id" };
  // 获取机构树数据
  const treeData = ref();
  //  默认展开二级树的节点id
  const defaultExpandedKeys = ref([]);
  const emit = defineEmits({ onBack: null });
  const tableData = ref([]);
  const selectedData = ref([]);
  const recordIds = ref();
  const props = defineProps(["radioModel", "dataIsConverterFlw", "orgTreeApi", "userPageApi", "checkedUserListApi"]);
  // 是否是单选
  const radioModel = props.radioModel || false;
  // 数据是否转换成工作流格式
  const dataIsConverterFlw = props.dataIsConverterFlw || false;
  // 分页相关
  const current = ref(0); // 当前页数
  const pageSize = ref(20); // 每页条数
  const total = ref(0); // 数据总数

  // 打开弹框
  const showUserPlusModal = (ids = []) => {
    visible.value = true;
    if (dataIsConverterFlw) {
      ids = goDataConverter(ids);
    }
    recordIds.value = ids;
    // 加载机构树
    if (props.orgTreeApi) {
      // 获取机构树
      props.orgTreeApi().then((data) => {
        cardLoading.value = false;
        if (data !== null) {
          treeData.value = data;
          // 默认展开2级
          treeData.value.forEach((item) => {
            // 因为0的顶级
            if (item.parentId === "0") {
              defaultExpandedKeys.value.push(item.id);
              // 取到下级ID
              if (item.children) {
                item.children.forEach((items) => {
                  defaultExpandedKeys.value.push(items.id);
                });
              }
            }
          });
        }
      });
    }
    searchFormState.value.size = pageSize.value;
    loadData();
    if (props.checkedUserListApi) {
      if (isEmpty(recordIds.value)) {
        return;
      }
      const param = {
        idList: recordIds.value
      };
      selectedTableListLoading.value = true;
      props
        .checkedUserListApi(param)
        .then((data) => {
          selectedData.value = data;
        })
        .finally(() => {
          selectedTableListLoading.value = false;
        });
    }
  };
  // 查询主表格数据
  const loadData = () => {
    pageLoading.value = true;
    props
      .userPageApi(searchFormState.value)
      .then((data) => {
        current.value = data.current;
        // pageSize.value = data.size
        total.value = data.total;
        // 重置、赋值
        tableData.value = [];
        tableRecordNum.value = 0;
        tableData.value = data.records;
        if (data.records) {
          tableRecordNum.value = data.records.length;
        } else {
          tableRecordNum.value = 0;
        }
      })
      .finally(() => {
        pageLoading.value = false;
      });
  };
  // pageSize改变回调分页事件
  const paginationChange = (page, pageSize) => {
    searchFormState.value.current = page;
    searchFormState.value.size = pageSize;
    loadData();
  };
  const judge = () => {
    if (radioModel && selectedData.value.length > 0) {
      return false;
    }
    return true;
  };
  // 添加记录
  const addRecord = (record) => {
    if (!judge()) {
      message.warning("只可选择一条");
      return;
    }
    const selectedRecord = selectedData.value.filter((item) => item.id === record.id);
    if (selectedRecord.length === 0) {
      selectedData.value.push(record);
    } else {
      message.warning("该记录已存在");
    }
  };
  // 添加全部
  const addAllPageRecord = () => {
    let newArray = selectedData.value.concat(tableData.value);
    let list = [];
    for (let item1 of newArray) {
      let flag = true;
      for (let item2 of list) {
        if (item1.id === item2.id) {
          flag = false;
        }
      }
      if (flag) {
        list.push(item1);
      }
    }
    selectedData.value = list;
  };
  // 删减记录
  const delRecord = (record) => {
    remove(selectedData.value, (item) => item.id === record.id);
  };
  // 删减记录
  const delAllRecord = () => {
    selectedData.value = [];
  };
  // 点击树查询
  const treeSelect = (selectedKeys) => {
    searchFormState.value.current = 0;
    if (selectedKeys.length > 0) {
      searchFormState.value.orgId = selectedKeys.toString();
    } else {
      delete searchFormState.value.orgId;
    }
    loadData();
  };
  // 确定
  const handleOk = () => {
    const value = [];
    selectedData.value.forEach((item) => {
      console.log(item);
      const obj = {
        id: item.id,
        name: item.name,
        account: item.account,
        empNo: item.empNo,
        orgId: item.orgId
      };
      value.push(obj);
    });
    // 判断是否做数据的转换为工作流需要的
    if (dataIsConverterFlw) {
      emit("onBack", outDataConverter(value));
    } else {
      emit("onBack", value);
    }
    handleClose();
  };
  // 重置
  const reset = () => {
    delete searchFormState.value.searchKey;
    loadData();
  };
  const handleClose = () => {
    searchFormState.value = {};
    tableRecordNum.value = 0;
    tableData.value = [];
    current.value = 0;
    pageSize.value = 20;
    total.value = 0;
    selectedData.value = [];
    visible.value = false;
  };

  // 数据进入后转换
  const goDataConverter = (data) => {
    const resultData = [];
    if (data.length > 0) {
      const values = data[0].value.split(",");
      if (JSON.stringify(values) !== '[""]') {
        for (let i = 0; i < values.length; i++) {
          resultData.push(values[i]);
        }
      }
    }
    return resultData;
  };
  // 数据出口转换器
  const outDataConverter = (data) => {
    const obj = {};
    let label = "";
    let value = "";
    for (let i = 0; i < data.length; i++) {
      if (data.length === i + 1) {
        label = label + data[i].name;
        value = value + data[i].id;
      } else {
        label = label + data[i].name + ",";
        value = value + data[i].id + ",";
      }
    }
    obj.key = "USER";
    obj.label = label;
    obj.value = value;
    obj.extJson = "";
    return obj;
  };
  defineExpose({
    showUserPlusModal
  });
</script>



```

```css
<style lang="less" scoped>
  .selectorTreeDiv {
    max-height: 500px;
    overflow: auto;
  }

  .cardTag {
    margin-left: 10px;
  }

  .primarySele {
    margin-right: 10px;
  }

  .ant-form-item {
    margin-bottom: 0 !important;
  }

  .user-table {
    overflow: auto;
    max-height: 450px;
  }
</style>
```

> index.vue

```html
<template>
  <a-button type="link" style="padding-left: 0px" @click="openSelector(formData.id)">选择</a-button>
  <a-tag v-if="formData.name" color="orange" closable @close="closeUserTag"> {{ formData.name }} </a-tag>

  <user-selector-plus
    ref="UserSelectorPlus"
    :org-tree-api="selectorApiFunction.orgTreeApi"
    :user-page-api="selectorApiFunction.userPageApi"
    :checked-user-list-api="selectorApiFunction.checkedUserListApi"
    @onBack="userBack"
  />
</template>
```

```js
<script lang="ts">
import bizOrgApi from '@/api/biz/bizOrgApi'
import userCenterApi from '@/api/sys/userCenterApi' 
import userSelectorPlus from '@/components/Selector/userSelectorPlus.vue'

let UserSelectorPlus = ref()
// 表单数据
const formData = ref({})

// 打开人员选择器，选择主管
const openSelector = (id) => {
	let checkedUserIds = []
	checkedUserIds.push(id)
	UserSelectorPlus.value.showUserPlusModal(checkedUserIds)
}
// 人员选择器回调
const userBack = (value) => {
	if (value.length > 0) {
		formData.value.id = value[0].id
		formData.value.name = value[0].name
	} else {
		formData.value.id = ''
		formData.value.name = ''
	}
}
// 通过小标签删除主管
const closeUserTag = () => {
	formData.value.id = ''
	formData.value.name = ''
}

// 传递设计器需要的API
const selectorApiFunction = {
	orgTreeApi: (param) => {
		return bizOrgApi.orgTreeSelector(param).then((data) => {
			return Promise.resolve(data)
		})
	},
	userPageApi: (param) => {
		return bizOrgApi.orgUserSelector(param).then((data) => {
			return Promise.resolve(data)
		})
	},
	checkedUserListApi: (param) => {
		return userCenterApi.userCenterGetUserListByIdList(param).then((data) => {
			return Promise.resolve(data)
		})
	}
}
</script>
```
