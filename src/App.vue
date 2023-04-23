<script setup lang="ts">
import { ref, nextTick } from "vue";
import { ElNotification } from "element-plus";
import { ipcRenderer } from "electron";

const dimensionMap: Map<string, string> = new Map();
dimensionMap.set("15分钟", "quarter_hour");
dimensionMap.set("30分钟", "half_hour");
dimensionMap.set("1小时", "one_hour");
dimensionMap.set("1天", "day");
dimensionMap.set("1周", "week");
dimensionMap.set("1月", "month");
dimensionMap.set("1年", "year");

const storeTypeMap: Map<string, string> = new Map();
storeTypeMap.set("增量", "增量");
storeTypeMap.set("全量", "全量");
storeTypeMap.set("拉链", "拉链");

const dbTypeMap: Map<string, string> = new Map();
dbTypeMap.set("hive", "hive");
dbTypeMap.set("postgres", "postgres");

const systemTypeMap: Map<string, string> = new Map();
systemTypeMap.set("数仓", "数仓");
systemTypeMap.set("子系统", "子系统");

const valueTypeMap: Map<string, string> = new Map();
valueTypeMap.set("string", "string");
valueTypeMap.set("bigint", "bigint");
valueTypeMap.set("int", "int");
valueTypeMap.set("tinyint", "tinyint");
valueTypeMap.set("double", "double");
valueTypeMap.set("boolean", "boolean");
valueTypeMap.set("timestamp", "timestamp");
valueTypeMap.set("array", "array");

const boolMap: Map<string, string> = new Map();
boolMap.set("true", "true");
boolMap.set("false", "false");

interface Setting {
  cnName: string,
  enName: string,
  dimension: string, // 维度
  period: string, // 周期
  storeType: string, // 存储类型
  dbType: string, // 数据库类型
  systemType: string, // 所属系统
  tableComments: string, // 说明
}

const setting = ref<Setting>({
  cnName: "表1",
  enName: "table1",
  dimension: "1天",
  period: "1天",
  storeType: "增量",
  dbType: "hive",
  systemType: "数仓",
  tableComments: "",
});

interface Column {
  enName: string,
  cnName: string,
  type: string,
  isPartitonKey: string,
  isClusterKey: string,
  needSort: string,
  comments: string
}

const columns = ref<Column[]>([{
  enName: "columnName",
  cnName: "列名",
  type: "string",
  isPartitonKey: "false",
  isClusterKey: "false",
  needSort: "false",
  comments: ""
}]);

async function addRow() {
  let newColumn = {
    enName: "",
    cnName: "",
    type: "",
    isNullable: "true",
    isPartitonKey: "false",
    isClusterKey: "false",
    needSort: "false",
    comments: ""
  };
  columns.value.push(newColumn);
  scrollBottom();
}

const selectedRows = ref<Column[]>([]);

async function onTableRowSelected(selection: any, row: Column) {
  selectedRows.value = selection;
}

async function removeSelectedRows() {
  columns.value = columns.value.filter((column: Column) => !selectedRows.value.find(selected => selected === column));
  selectedRows.value = [];
}

function scrollBottom() {
  nextTick(() => { document.documentElement.scrollTop = 999999999; })
}

function padZero(num: number) {
  if (num < 10) {
    return `0${num}`;
  } else {
    return `${num}`;
  }
}

function commentsMerge(column: Column): string {
  let comments: string[] = [];
  if (column.cnName) {
    comments.push(column.cnName);
  }
  if (column.comments) {
    comments.push(column.comments);
  }
  return comments.join(", ");
}

const sql = ref("");

function generateDDL() {
  sql.value = "";
  const valideColumns = columns.value.filter(column => column.enName !== "");
  if (valideColumns.length < columns.value.length) {
    nextTick(() => {
      ElNotification({
        type: "success",
        message: "已过滤掉无效数据！",
        position: "top-right"
      });
    });
  }
  columns.value = valideColumns;

  let map = new Map<string, Column>();
  for (let i = columns.value.length - 1; i >= 0; i--) {
    map.set(columns.value[i].enName, columns.value[i]);
  }
  const distincted = Array.from(map.keys());
  if (distincted.length < columns.value.length) {
    nextTick(() => {
      ElNotification({
        type: "success",
        message: "已过滤掉重复数据！",
        position: "top-right"
      });
    });
  }
  columns.value = Array.from(map.values()).reverse();

  if (columns.value.length === 0) {
    nextTick(() => {
      ElNotification({
        type: "error",
        message: "数据为空！",
        position: "top-right"
      });
    });
    return;
  }

  let dateTime = new Date();
  let today = `${padZero(dateTime.getFullYear())}-${padZero(dateTime.getMonth() + 1)}-${padZero(dateTime.getDate())}`;
  let sqlInfo = `-- 表中文名：${setting.value.cnName}
-- 表英文名：${setting.value.enName}
-- 统计维度：${setting.value.dimension}
-- 存储周期：${setting.value.period}
-- 存储方式：${setting.value.storeType}
-- 数据库类型：${setting.value.dbType}
-- 所属系统：${setting.value.systemType}
-- 说明：${setting.value.tableComments}
-- 创建时间：${today}`;

  let partitionColumns = columns.value.filter(column => column.isPartitonKey === "true");
  let partitionColumnString = partitionColumns.map(column => `${column.enName} ${column.type.toUpperCase()} COMMENT '${commentsMerge(column)}'`).join(", ");
  let partition = `PARTITIONED BY (${partitionColumnString})`

  let normalColumns = columns.value.filter(column => column.isPartitonKey === "false");
  if (normalColumns.length === 0) {
    nextTick(() => {
      ElNotification({
        type: "error",
        message: "无效数据，请完善列信息后再试！",
        position: "top-right"
      });
    });
    return;
  }
  let normalColumnString = normalColumns.map(column => {
    return `  ${column.enName} ${column.type.toUpperCase()} COMMENT '${commentsMerge(column)}'`;
  }).join(",\r\n");

  let clusterColumns = columns.value.filter(column => column.isClusterKey === "true");
  let clusterColumnString = clusterColumns.map(column => `${column.enName}`).join(", ");
  let cluster = `CLUSTERED BY(${clusterColumnString})`

  let sortColumns = columns.value.filter(column => column.needSort === "true");
  let sortColumnString = sortColumns.map(column => `${column.enName} ASC`).join(", ");
  let sort = `SORTED BY(${sortColumnString})`

  const tableComments: string[] = [];
  if (setting.value.cnName) {
    tableComments.push(setting.value.cnName);
  }
  if (setting.value.tableComments) {
    tableComments.push(setting.value.tableComments);
  }
  const tableCommentString = tableComments.join(", ");

  let sqlList: string[] = [
    sqlInfo,
    `DROP TABLE IF EXISTS ${setting.value.enName};`,
    `CREATE TABLE ${setting.value.enName} (`,
    normalColumnString,
    tableCommentString ? `) COMMENT '${tableCommentString}'` : `)`,
  ];
  if (partitionColumns.length > 0) {
    sqlList.push(partition);
  }
  if (clusterColumns.length > 0) {
    let innerList = [cluster];
    if (sortColumns.length > 0) {
      innerList.push(sort);
    }
    innerList.push("INTO 8 BUCKETS");
    sqlList.push(innerList.join(" "));
  }
  sqlList.push("ROW FORMAT DELIMITED FIELDS TERMINATED BY '\\t'");
  sqlList.push("STORED AS ORC;");
  sql.value = sqlList.join("\r\n");
  document.querySelector(".ddl")!.innerHTML = sql.value;

  nextTick(() => {
    ElNotification({
      type: "success",
      message: "DDL 生成成功！",
      position: "top-right"
    });
  });
}

async function copyDDl() {
  await navigator.clipboard.writeText(sql.value);
  ElNotification({
    type: "success",
    message: "复制成功！",
    position: "top-right"
  });
}

async function exportDDL() {
  const fileData = {
    fileName: `${setting.value.cnName || setting.value.enName}-${setting.value.dbType}.sql`,
    datas: sql.value
  }

  if (!fileData.datas) {
    nextTick(() => {
      ElNotification({
        type: "error",
        message: "暂无数据导出！",
        position: "top-right"
      });
    });
    return;
  }

  ipcRenderer.send("export-ddl", fileData);
}

ipcRenderer.on("export-ddl-success", function () {
  nextTick(() => {
    ElNotification({
      type: "success",
      message: "导出成功！",
      position: "top-right"
    });
  });
});

ipcRenderer.on("export-ddl-failed", function (event, error: string) {
  nextTick(() => {
    ElNotification({
      type: "error",
      message: `导出失败！${error}`,
      position: "top-right"
    });
  });
});

ipcRenderer.on("export-ddl-canceled", function (event, error: string) {
  nextTick(() => {
    ElNotification({
      type: "warning",
      message: `导出已取消！`,
      position: "top-right"
    });
  });
});

async function columnEnNameChange(name: string, row: Column) {
  const existed = columns.value.filter(col => col.enName === name).length > 1;
  if (existed) {
    nextTick(() => {
      ElNotification({
        type: "error",
        message: "字段已存在！",
        position: "top-right"
      });
    });
  }
}

</script>

<template>
  <el-row class="row">
    <el-text class="label">表中文名称</el-text>
    <el-input class="input" name="cnName" v-model="setting.cnName" clearable autocomplete="on" placeholder="表中文名称" />
  </el-row>
  <el-row class="row">
    <el-text class="label">表英文名称 *</el-text>
    <el-input class="input" name="enName" v-model="setting.enName" clearable autocomplete="on" placeholder="表英文名称" />
  </el-row>
  <el-row class="row">
    <el-text class="label">统计维度</el-text>
    <el-select class="input" name="dimension" v-model="setting.dimension" placeholder="统计维度">
      <el-option v-for="[key, value] in dimensionMap.entries()" :key="value" :label="key" :value="key" />
    </el-select>
  </el-row>
  <el-row class="row">
    <el-text class="label">存储周期</el-text>
    <el-select class="input" name="period" v-model="setting.period" placeholder="存储周期">
      <el-option v-for="[key, value] in dimensionMap.entries()" :key="value" :label="key" :value="key" />
    </el-select>
  </el-row>
  <el-row class="row">
    <el-text class="label">存储方式</el-text>
    <el-select class="input" name="storeType" v-model="setting.storeType" placeholder="存储方式">
      <el-option v-for="[key, value] in storeTypeMap.entries()" :key="value" :label="key" :value="key" />
    </el-select>
  </el-row>
  <el-row class="row">
    <el-text class="label">数据库类型</el-text>
    <el-select class="input" name="dbType" v-model="setting.dbType" placeholder="数据库类型">
      <el-option v-for="[key, value] in dbTypeMap.entries()" :key="value" :label="key" :value="key" />
    </el-select>
  </el-row>
  <el-row class="row">
    <el-text class="label">所属系统</el-text>
    <el-select class="input" name="systemType" v-model="setting.systemType" placeholder="所属系统">
      <el-option v-for="[key, value] in systemTypeMap.entries()" :key="value" :label="key" :value="key" />
    </el-select>
  </el-row>
  <el-row class="row">
    <el-text class="label">说明</el-text>
    <el-input class="input" name="tableComments" v-model="setting.tableComments" clearable autocomplete="on"
      placeholder="说明" />
  </el-row>

  <el-table empty-text="尚未添加数据" :data="columns" @select="onTableRowSelected" @select-all="onTableRowSelected">
    <el-table-column type="index" width="55" />
    <el-table-column label="字段名称 *">
      <template #default="scope">
        <el-input v-model="scope.row.enName" placeholder="字段名称" clearable
          @blur="columnEnNameChange(scope.row.enName, scope.row)"></el-input>
      </template>
    </el-table-column>
    <el-table-column label="字段中文名称">
      <template #default="scope">
        <el-input v-model="scope.row.cnName" placeholder="字段中文名称" clearable></el-input>
      </template>
    </el-table-column>
    <el-table-column label="字段类型">
      <template #default="scope">
        <el-select v-model="scope.row.type" placeholder="字段类型">
          <el-option v-for="[key, value] in valueTypeMap.entries()" :key="value" :label="key" :value="key" />
        </el-select>
      </template>
    </el-table-column>
    <el-table-column label="是否分区键">
      <template #default="scope">
        <el-select v-model="scope.row.isPartitonKey" placeholder="是否分区键">
          <el-option v-for="[key, value] in boolMap.entries()" :key="value" :label="key" :value="key" />
        </el-select>
      </template>
    </el-table-column>
    <el-table-column label="是否分桶键">
      <template #default="scope">
        <el-select v-model="scope.row.isClusterKey" placeholder="是否分桶键">
          <el-option v-for="[key, value] in boolMap.entries()" :key="value" :label="key" :value="key" />
        </el-select>
      </template>
    </el-table-column>
    <el-table-column label="是否排序">
      <template #default="scope">
        <el-select v-model="scope.row.needSort" placeholder="是否排序" :disabled="scope.row.isClusterKey === 'false'">
          <el-option v-for="[key, value] in boolMap.entries()" :key="value" :label="key" :value="key" />
        </el-select>
      </template>
    </el-table-column>
    <el-table-column label="备注">
      <template #default="scope">
        <el-input v-model="scope.row.comments" placeholder="备注" clearable></el-input>
      </template>
    </el-table-column>
    <el-table-column type="selection" width="55" />
  </el-table>

  <el-row justify="center">
    <el-button @click="addRow">添加字段</el-button>
    <el-button @click="generateDDL" :disabled="columns.length === 0">生成 DDL</el-button>
    <el-button @click="copyDDl" v-if="!!sql">复制 DDL</el-button>
    <el-button @click="exportDDL" v-if="!!sql">导出 DDL</el-button>
    <el-button @click="removeSelectedRows" :disabled="selectedRows.length === 0">删除选中</el-button>
  </el-row>

  <pre class="ddl" v-show="!!sql"></pre>
</template>

<style>
.el-row {
  margin-top: 10px;
  user-select: none;
}

.row {
  display: flex;
}

.row>.label {
  width: 100px;
  display: flex;
  align-items: center;
  justify-content: right;
  padding-right: 10px;
}

.row>.input {
  width: 300px;
  display: flex;
  align-items: center;
  justify-content: left;
}

.el-table {
  margin-top: 20px;
  user-select: none;
}

.el-select>div {
  width: 100%;
}

.ddl {
  padding: 10px;
  width: 70%;
  margin: 10px auto;
  border-radius: 5px;
  background-color: #eee;
}
</style>
