<template>
  <div class="table-list">
    <el-page-header @back="$router.go(-1)" title="返回" content="数据表管理" />

    <div class="table-list-content">
      <div class="table-list-header">
        <h2>数据表列表</h2>
        <el-button type="primary" @click="$router.push('/create-table')">
          创建新表
        </el-button>
      </div>

      <el-table
        :data="tables"
        style="width: 100%"
        v-loading="loading"
        border
        stripe
      >
        <el-table-column prop="id" label="ID" width="80" />
        <el-table-column prop="name" label="表名" min-width="150" />
        <el-table-column prop="description" label="描述" min-width="200" />
        <el-table-column
          prop="field_count"
          label="字段数"
          width="100"
          align="center"
        />
        <el-table-column
          prop="api_route_count"
          label="API数"
          width="100"
          align="center"
        />
        <el-table-column prop="created_at" label="创建时间" width="180">
          <template #default="scope">
            {{ formatDate(scope.row.created_at) }}
          </template>
        </el-table-column>
        <el-table-column label="操作" width="250" fixed="right">
          <template #default="scope">
            <el-button size="small" @click="viewTableDetail(scope.row.id)">
              查看
            </el-button>
            <el-button
              size="small"
              type="warning"
              @click="viewTableAPIs(scope.row.id)"
            >
              API
            </el-button>
            <el-popconfirm
              title="确定删除该表吗？所有关联的API也将被删除！"
              @confirm="deleteTable(scope.row.id)"
              confirm-button-type="danger"
            >
              <template #reference>
                <el-button size="small" type="danger">删除</el-button>
              </template>
            </el-popconfirm>
          </template>
        </el-table-column>
      </el-table>

      <div v-if="tables.length === 0 && !loading" class="empty-state">
        <el-empty description="暂无数据表">
          <el-button type="primary" @click="$router.push('/create-table')">
            创建第一个表
          </el-button>
        </el-empty>
      </div>
    </div>

    <!-- 表详情对话框 -->
    <el-dialog
      v-model="detailDialogVisible"
      title="表详情"
      width="70%"
      destroy-on-close
    >
      <div v-loading="detailLoading">
        <template v-if="selectedTable">
          <h3>{{ selectedTable.name }}</h3>
          <p>{{ selectedTable.description }}</p>

          <div class="table-sql">
            <h4>创建SQL</h4>
            <pre>{{ selectedTable.sql_create }}</pre>
          </div>

          <div class="table-fields">
            <h4>字段列表</h4>
            <el-table :data="tableFields" border>
              <el-table-column prop="name" label="字段名" width="150" />
              <el-table-column prop="type" label="类型" width="120" />
              <el-table-column prop="length" label="长度" width="80" />
              <el-table-column
                prop="description"
                label="描述"
                min-width="200"
              />
              <el-table-column label="约束" width="200">
                <template #default="scope">
                  <el-tag
                    v-if="scope.row.primary_key"
                    size="small"
                    type="success"
                    >主键</el-tag
                  >
                  <el-tag
                    v-if="scope.row.auto_increment"
                    size="small"
                    type="warning"
                    >自增</el-tag
                  >
                  <el-tag v-if="scope.row.unique_field" size="small" type="info"
                    >唯一</el-tag
                  >
                  <el-tag v-if="!scope.row.nullable" size="small" type="danger"
                    >非空</el-tag
                  >
                </template>
              </el-table-column>
              <el-table-column
                prop="default_value"
                label="默认值"
                width="120"
              />
            </el-table>
          </div>
        </template>
      </div>
    </el-dialog>

    <!-- API列表对话框 -->
    <el-dialog
      v-model="apiDialogVisible"
      title="API路由列表"
      width="80%"
      destroy-on-close
    >
      <div v-loading="apiLoading">
        <template v-if="selectedTable">
          <h3>表 {{ selectedTable.name }} 的API路由</h3>

          <div class="api-actions">
            <el-button
              type="primary"
              @click="exportApiDocs"
              :loading="exportLoading"
              :disabled="tableApis.length === 0"
            >
              导出接口文档
            </el-button>
            <el-button
              type="success"
              @click="previewApiDocs"
              :disabled="tableApis.length === 0"
            >
              预览文档
            </el-button>
          </div>

          <el-table :data="tableApis" border>
            <el-table-column prop="id" label="ID" width="80" />
            <el-table-column prop="name" label="名称" min-width="150" />
            <el-table-column prop="path" label="路径" min-width="200" />
            <el-table-column prop="method" label="方法" width="100" />
            <el-table-column
              prop="crud_operation"
              label="操作类型"
              width="120"
            />
            <el-table-column prop="is_public" label="公开" width="80">
              <template #default="scope">
                <el-tag :type="scope.row.is_public ? 'success' : 'info'">
                  {{ scope.row.is_public ? "是" : "否" }}
                </el-tag>
              </template>
            </el-table-column>
            <el-table-column label="操作" width="120" fixed="right">
              <template #default="scope">
                <el-button size="small" @click="testApi(scope.row)">
                  测试
                </el-button>
              </template>
            </el-table-column>
          </el-table>
        </template>
      </div>
    </el-dialog>

    <!-- 添加页脚组件 -->
    <AppFooter />

    <!-- API文档预览对话框 -->
    <el-dialog
      v-model="previewDialogVisible"
      title="API文档预览"
      width="90%"
      destroy-on-close
      fullscreen
    >
      <div v-loading="previewLoading" class="api-preview-content">
        <div class="preview-actions">
          <el-button
            type="primary"
            @click="exportApiDocsFromPreview"
            size="small"
          >
            导出文档
          </el-button>
        </div>
        <div class="markdown-preview" v-html="markdownHtml"></div>
      </div>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
import { ElMessage, ElMessageBox } from "element-plus";
import { useRouter } from "vue-router";
import AppFooter from "~/components/AppFooter.vue";

// 路由器
const router = useRouter();

// 加载状态
const loading = ref(true);
const detailLoading = ref(false);
const apiLoading = ref(false);
const exportLoading = ref(false);
const previewLoading = ref(false);

// 表格数据
const tables = ref<any[]>([]);

// 选中的表
const selectedTable = ref<any>(null);
const tableFields = ref<any[]>([]);
const tableApis = ref<any[]>([]);

// 对话框可见性
const detailDialogVisible = ref(false);
const apiDialogVisible = ref(false);
const previewDialogVisible = ref(false);

// 预览相关
const markdownContent = ref("");
const markdownHtml = ref("");

// 定义响应数据类型
interface ApiResponse<T> {
  success: boolean;
  statusMessage?: string;
  [key: string]: any;
}

interface TableResponse extends ApiResponse<any> {
  table: any;
  fields?: any[];
  api_routes?: any[];
}

// 格式化日期
const formatDate = (dateString: string) => {
  if (!dateString) return "-";
  const date = new Date(dateString);
  return date.toLocaleString();
};

// 加载表列表
const loadTables = async () => {
  loading.value = true;
  try {
    const response = await fetch("/api/admin/tables");

    if (!response.ok) {
      throw new Error("获取表列表失败");
    }

    const data = (await response.json()) as ApiResponse<{ tables: any[] }>;

    if (data.success) {
      tables.value = data.tables || [];
    } else {
      throw new Error(data.statusMessage || "获取表列表失败");
    }
  } catch (error: any) {
    ElMessage.error({
      message: error.message || "获取表列表失败",
      duration: 5000,
    });
  } finally {
    loading.value = false;
  }
};

// 查看表详情
const viewTableDetail = async (tableId: number) => {
  detailLoading.value = true;
  detailDialogVisible.value = true;

  try {
    const response = await fetch(`/api/admin/tables/${tableId}`);

    if (!response.ok) {
      throw new Error("获取表详情失败");
    }

    const data = (await response.json()) as TableResponse;

    if (data.success) {
      selectedTable.value = data.table;
      tableFields.value = data.fields || [];
    } else {
      throw new Error(data.statusMessage || "获取表详情失败");
    }
  } catch (error: any) {
    ElMessage.error({
      message: error.message || "获取表详情失败",
      duration: 5000,
    });
    detailDialogVisible.value = false;
  } finally {
    detailLoading.value = false;
  }
};

// 查看表API
const viewTableAPIs = async (tableId: number) => {
  apiLoading.value = true;
  apiDialogVisible.value = true;

  try {
    const response = await fetch(`/api/admin/tables/${tableId}`);

    if (!response.ok) {
      throw new Error("获取表API失败");
    }

    const data = (await response.json()) as TableResponse;

    if (data.success) {
      selectedTable.value = data.table;
      tableApis.value = data.api_routes || [];
      tableFields.value = data.fields || [];
    } else {
      throw new Error(data.statusMessage || "获取表API失败");
    }
  } catch (error: any) {
    ElMessage.error({
      message: error.message || "获取表API失败",
      duration: 5000,
    });
    apiDialogVisible.value = false;
  } finally {
    apiLoading.value = false;
  }
};

// 测试API
const testApi = (api: any) => {
  router.push({
    path: "/api-tester",
    query: { path: api.path, method: api.method },
  });
};

// 删除表
const deleteTable = async (tableId: number) => {
  try {
    const response = await fetch(`/api/admin/tables/${tableId}`, {
      method: "DELETE",
    });

    if (!response.ok) {
      throw new Error("删除表失败");
    }

    const data = (await response.json()) as ApiResponse<any>;

    if (data.success) {
      ElMessage.success("表删除成功");
      loadTables(); // 重新加载表列表
    } else {
      throw new Error(data.statusMessage || "删除表失败");
    }
  } catch (error: any) {
    ElMessage.error({
      message: error.message || "删除表失败",
      duration: 5000,
    });
  }
};

// 导出接口文档
const exportApiDocs = async () => {
  exportLoading.value = true;
  try {
    // 输出调试信息，查看API数据结构
    console.log("导出文档 - 表信息:", selectedTable.value);
    console.log("导出文档 - API列表:", tableApis.value);
    console.log("导出文档 - 字段列表:", tableFields.value);

    // 检查API参数信息
    tableApis.value.forEach((api, index) => {
      console.log(`API ${index + 1} - ${api.name} 参数:`, api.params);
      if (api.params) {
        try {
          const parsedParams = JSON.parse(api.params);
          console.log(`API ${index + 1} - 解析后参数:`, parsedParams);
        } catch (e) {
          console.error(`API ${index + 1} - 参数解析失败:`, e);
        }
      }
    });

    // 生成Markdown文档
    const content = generateApiDocs();
    markdownContent.value = content;

    // 下载文档
    downloadMarkdown(
      content,
      `${selectedTable.value?.name || "table"}-api-docs.md`
    );

    ElMessage.success("接口文档导出成功");
  } catch (error: any) {
    ElMessage.error({
      message: error.message || "导出接口文档失败",
      duration: 5000,
    });
  } finally {
    exportLoading.value = false;
  }
};

// 预览API文档
const previewApiDocs = async () => {
  previewLoading.value = true;
  previewDialogVisible.value = true;

  try {
    // 生成Markdown文档
    const content = generateApiDocs();
    markdownContent.value = content;

    // 转换Markdown为HTML (使用简单的方法，实际项目中可以使用marked.js等库)
    markdownHtml.value = convertMarkdownToHtml(content);
  } catch (error: any) {
    ElMessage.error({
      message: error.message || "预览接口文档失败",
      duration: 5000,
    });
  } finally {
    previewLoading.value = false;
  }
};

// 从预览对话框导出文档
const exportApiDocsFromPreview = () => {
  if (markdownContent.value) {
    downloadMarkdown(
      markdownContent.value,
      `${selectedTable.value?.name || "table"}-api-docs.md`
    );
    ElMessage.success("接口文档导出成功");
  } else {
    ElMessage.error("文档内容为空，无法导出");
  }
};

// 将Markdown转换为HTML的简单实现
const convertMarkdownToHtml = (markdown: string): string => {
  if (!markdown) return "";

  let html = markdown;

  // 转换代码块（带语法高亮）
  html = html.replace(/```(\w+)\n([\s\S]*?)```/gm, (match, language, code) => {
    // 对代码进行HTML转义
    const escapedCode = code
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;")
      .replace(/'/g, "&#039;");

    return `<pre class="language-${language}"><code class="language-${language}">${escapedCode}</code></pre>`;
  });

  // 转换没有指定语言的代码块
  html = html.replace(/```\n([\s\S]*?)```/gm, (match, code) => {
    // 对代码进行HTML转义
    const escapedCode = code
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;")
      .replace(/'/g, "&#039;");

    return `<pre><code>${escapedCode}</code></pre>`;
  });

  // 转换内联代码
  html = html.replace(/`([^`]+)`/gm, (match, code) => {
    // 对代码进行HTML转义
    const escapedCode = code
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;")
      .replace(/'/g, "&#039;");

    return `<code>${escapedCode}</code>`;
  });

  // 转换标题
  html = html.replace(/^# (.*$)/gm, "<h1>$1</h1>");
  html = html.replace(/^## (.*$)/gm, "<h2>$1</h2>");
  html = html.replace(/^### (.*$)/gm, "<h3>$1</h3>");
  html = html.replace(/^#### (.*$)/gm, "<h4>$1</h4>");

  // 转换表格
  const tableRegex = /\|(.+)\|[\r\n]\|([-]+)\|[\r\n]([\s\S]*?)(?=[\r\n]{2}|$)/g;
  html = html.replace(tableRegex, function (match, headers, separators, rows) {
    const headerCells = headers
      .split("|")
      .map((cell: string) => `<th>${cell.trim()}</th>`)
      .join("");
    const tableRows = rows
      .split("\n")
      .map((row: string) => {
        if (!row.trim()) return "";
        const cells = row
          .split("|")
          .map((cell: string) => `<td>${cell.trim()}</td>`)
          .join("");
        return `<tr>${cells}</tr>`;
      })
      .join("");

    return `<table border="1"><thead><tr>${headerCells}</tr></thead><tbody>${tableRows}</tbody></table>`;
  });

  // 转换列表
  html = html.replace(/^\- (.*$)/gm, "<li>$1</li>");
  html = html.replace(/(<li>.*<\/li>\n<li>.*<\/li>)/gm, "<ul>$1</ul>");

  // 转换加粗
  html = html.replace(/\*\*(.*?)\*\*/gm, "<strong>$1</strong>");

  // 转换斜体
  html = html.replace(/\*(.*?)\*/gm, "<em>$1</em>");

  // 转换链接
  html = html.replace(/\[(.*?)\]\((.*?)\)/gm, '<a href="$2">$1</a>');

  // 转换段落（确保不会转换已经是HTML标签的内容）
  html = html.replace(/^(?!<[a-z])(.*$)/gm, "<p>$1</p>");

  // 处理换行
  html = html.replace(/\n\n/gm, "<br><br>");

  return html;
};

// 生成API文档Markdown内容
const generateApiDocs = (): string => {
  if (!selectedTable.value || tableApis.value.length === 0) {
    throw new Error("没有可用的API数据");
  }

  let markdown = "";

  // 获取基础URL
  const baseUrl = window.location.origin;

  // 添加标题
  markdown += `# ${selectedTable.value.name} 接口文档\n\n`;

  // 添加目录
  markdown += `## 目录\n\n`;
  markdown += `- [基础信息](#基础信息)\n`;
  markdown += `- [数据结构](#数据结构)\n`;
  markdown += `- [API接口](#api接口)\n`;
  tableApis.value.forEach((api, index) => {
    const apiName = api.name.replace(/\s+/g, "-");
    markdown += `  - [${index + 1}. ${api.name}](#${index + 1}-${apiName})\n`;
  });
  markdown += `- [附录](#附录)\n\n`;

  // 添加表描述
  if (selectedTable.value.description) {
    markdown += `## 表描述\n\n${selectedTable.value.description}\n\n`;
  }

  // 添加基础URL信息
  markdown += `## 基础信息\n\n`;
  markdown += `- **基础URL**: \`${baseUrl}\`\n`;
  markdown += `- **数据格式**: JSON\n`;
  markdown += `- **认证方式**: 公开API无需认证，非公开API需要认证\n`;
  markdown += `- **表名**: ${selectedTable.value.name}\n`;
  if (selectedTable.value.sql_create) {
    markdown += `- **创建SQL**:\n\n\`\`\`sql\n${selectedTable.value.sql_create}\n\`\`\`\n`;
  }
  markdown += `\n`;

  // 添加表字段信息
  markdown += `## 数据结构\n\n`;
  markdown += `### 表字段\n\n`;
  markdown += `| 字段名 | 类型 | 长度 | 描述 | 约束 | 默认值 |\n`;
  markdown += `| ------ | ---- | ---- | ---- | ---- | ------ |\n`;

  // 如果有字段信息，添加字段信息
  if (tableFields.value && tableFields.value.length > 0) {
    tableFields.value.forEach((field) => {
      const constraints = [];
      if (field.primary_key) constraints.push("主键");
      if (field.auto_increment) constraints.push("自增");
      if (field.unique_field) constraints.push("唯一");
      if (!field.nullable) constraints.push("非空");

      markdown += `| ${field.name} | ${field.type} | ${field.length || "-"} | ${
        field.description || "-"
      } | ${constraints.join(", ") || "-"} | ${field.default_value || "-"} |\n`;
    });
  } else {
    markdown += `| - | - | - | - | - | - |\n`;
  }

  markdown += `\n`;

  // 添加API信息
  markdown += `## API接口\n\n`;

  tableApis.value.forEach((api, index) => {
    const apiName = api.name.replace(/\s+/g, "-");
    markdown += `### ${index + 1}. ${api.name}\n\n`;
    markdown += `- **路径**: \`${api.path}\`\n`;
    markdown += `- **完整URL**: \`${baseUrl}${api.path}\`\n`;
    markdown += `- **方法**: \`${api.method}\`\n`;
    markdown += `- **类型**: ${
      getCrudOperationName(api.crud_operation) || "-"
    }\n`;
    markdown += `- **公开**: ${api.is_public ? "是" : "否"}\n`;

    if (api.description) {
      markdown += `- **描述**: ${api.description}\n`;
    }

    // 如果有SQL查询，添加SQL查询信息
    if (api.sql_query) {
      markdown += `\n#### SQL查询\n\n`;
      markdown += "```sql\n";
      markdown += api.sql_query;
      markdown += "\n```\n\n";
    }

    // 改进参数信息处理
    let hasParams = false;
    let parsedParams = {};

    // 如果有参数信息
    if (api.params) {
      try {
        // 尝试解析参数
        parsedParams = JSON.parse(api.params);

        // 检查是否有参数
        if (Object.keys(parsedParams).length > 0) {
          hasParams = true;
          markdown += `\n#### 请求参数\n\n`;
          markdown += `| 参数名 | 类型 | 必填 | 描述 |\n`;
          markdown += `| ------ | ---- | ---- | ---- |\n`;

          Object.entries(parsedParams).forEach(
            ([key, value]: [string, any]) => {
              const paramType = value.type || "string";
              const isRequired = value.required === true ? "是" : "否";
              const description = value.description || "-";

              markdown += `| ${key} | ${paramType} | ${isRequired} | ${description} |\n`;
            }
          );
        }
      } catch (e) {
        // 参数解析失败，添加错误提示
        console.error(`API ${api.name} 参数解析失败:`, e);
        markdown += `\n#### 请求参数\n\n`;
        markdown += `> 参数解析失败，原始参数字符串: \`${api.params}\`\n\n`;

        // 尝试提取参数信息，即使解析JSON失败
        if (typeof api.params === "string" && api.params.trim()) {
          hasParams = true;

          // 尝试使用正则表达式匹配参数信息
          const paramMatches = api.params.match(/"([^"]+)":/g);
          if (paramMatches && paramMatches.length > 0) {
            markdown += `\n可能的参数名称:\n\n`;
            paramMatches.forEach((match: string) => {
              const paramName = match.replace(/[":]|"/g, "");
              markdown += `- \`${paramName}\`\n`;
            });
            markdown += `\n`;
          }
        }
      }
    }

    if (!hasParams) {
      markdown += `\n#### 请求参数\n\n`;
      markdown += `> 此API没有请求参数\n\n`;
    } else {
      // 添加参数说明部分，只有在成功解析参数时才添加
      if (Object.keys(parsedParams).length > 0) {
        markdown += `\n#### 参数说明\n\n`;

        Object.entries(parsedParams).forEach(([key, value]: [string, any]) => {
          const paramType = value.type || "string";
          const isRequired = value.required === true ? "是" : "否";
          const description = value.description || "无描述";

          markdown += `**${key}**\n`;
          markdown += `- 类型: \`${paramType}\`\n`;
          markdown += `- 必填: ${isRequired}\n`;
          markdown += `- 描述: ${description}\n`;

          // 添加示例值
          let exampleValue = "";
          switch (paramType.toLowerCase()) {
            case "number":
              exampleValue = "1";
              break;
            case "boolean":
              exampleValue = "true";
              break;
            case "date":
              exampleValue = "2023-01-01";
              break;
            case "array":
              exampleValue = "[1, 2, 3]";
              break;
            case "object":
              exampleValue = '{ "key": "value" }';
              break;
            default:
              exampleValue = '"example"';
          }

          markdown += `- 示例值: \`${exampleValue}\`\n\n`;
        });
      }
    }

    // 添加请求示例
    markdown += `\n#### 请求示例\n\n`;

    // cURL示例
    markdown += `**cURL**\n\n`;
    markdown += "```bash\n";
    markdown += generateCurlExample(api);
    markdown += "\n```\n\n";

    // JavaScript示例
    markdown += `**JavaScript**\n\n`;
    markdown += "```javascript\n";
    markdown += generateJsExample(api);
    markdown += "\n```\n\n";

    // 添加返回示例
    markdown += `#### 返回示例\n\n`;
    markdown += "```json\n";
    markdown += generateResponseExample(api);
    markdown += "\n```\n\n";

    markdown += `---\n\n`;
  });

  // 添加生成时间
  markdown += `## 附录\n\n`;
  markdown += `- **文档生成时间**: ${new Date().toLocaleString()}\n`;
  markdown += `- **生成工具**: SQL to API 接口文档生成器\n`;
  markdown += `- **总API数量**: ${tableApis.value.length}\n`;
  markdown += `- **总字段数量**: ${tableFields.value.length}\n`;

  return markdown;
};

// 获取CRUD操作类型名称
const getCrudOperationName = (operation: string): string => {
  const map: Record<string, string> = {
    READ_ALL: "读取全部",
    READ_ONE: "读取单个",
    CREATE: "创建",
    UPDATE: "更新",
    DELETE: "删除",
    SEARCH: "搜索",
  };
  return map[operation] || operation || "";
};

// 下载Markdown文件
const downloadMarkdown = (content: string, filename: string): void => {
  // 创建Blob对象
  const blob = new Blob([content], { type: "text/markdown;charset=utf-8" });

  // 创建下载链接
  const url = URL.createObjectURL(blob);

  // 创建一个a标签用于下载
  const link = document.createElement("a");
  link.href = url;
  link.download = filename;

  // 添加到文档并模拟点击
  document.body.appendChild(link);
  link.click();

  // 清理
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
};

// 生成cURL请求示例
const generateCurlExample = (api: any): string => {
  const baseUrl = window.location.origin;
  const url = `${baseUrl}${api.path}`;
  let curl = `curl -X ${api.method} "${url}"`;

  // 添加内容类型头
  if (api.method !== "GET") {
    curl += ' -H "Content-Type: application/json"';
  }

  // 添加请求体
  if (api.method !== "GET" && api.params) {
    try {
      const params = JSON.parse(api.params);
      if (Object.keys(params).length > 0) {
        const exampleBody: Record<string, any> = {};

        // 为每个参数创建示例值
        Object.entries(params).forEach(([key, value]: [string, any]) => {
          // 根据参数类型生成示例值
          switch (value.type?.toLowerCase()) {
            case "number":
              exampleBody[key] = 1;
              break;
            case "boolean":
              exampleBody[key] = true;
              break;
            case "date":
              exampleBody[key] = "2023-01-01";
              break;
            case "array":
              exampleBody[key] = [1, 2, 3];
              break;
            case "object":
              exampleBody[key] = { key: "value" };
              break;
            default:
              exampleBody[key] = `示例${key}`;
          }
        });

        curl += ` -d '${JSON.stringify(exampleBody, null, 2)}'`;
      }
    } catch (e) {
      // 参数解析失败，添加一个基本的请求体示例
      console.error("cURL示例参数解析失败:", e);

      // 尝试从原始参数字符串中提取可能的参数名
      let exampleBody: Record<string, any> = {};

      if (typeof api.params === "string" && api.params.trim()) {
        const paramMatches = api.params.match(/"([^"]+)":/g);
        if (paramMatches && paramMatches.length > 0) {
          paramMatches.forEach((match: string) => {
            const paramName = match.replace(/[":]|"/g, "");
            exampleBody[paramName] = `示例${paramName}`;
          });
        }
      }

      // 如果无法提取参数，使用空对象
      if (Object.keys(exampleBody).length === 0) {
        exampleBody = { param: "value" };
      }

      curl += ` -d '${JSON.stringify(exampleBody, null, 2)}'`;
    }
  } else if (api.method === "GET" && api.params) {
    // 处理GET请求的查询参数
    try {
      const params = JSON.parse(api.params);
      if (Object.keys(params).length > 0) {
        const queryParams: string[] = [];

        // 为每个参数创建示例值
        Object.entries(params).forEach(([key, value]: [string, any]) => {
          let exampleValue: string;

          // 根据参数类型生成示例值
          switch (value.type?.toLowerCase()) {
            case "number":
              exampleValue = "1";
              break;
            case "boolean":
              exampleValue = "true";
              break;
            case "date":
              exampleValue = "2023-01-01";
              break;
            default:
              exampleValue = `example${key}`;
          }

          queryParams.push(`${key}=${encodeURIComponent(exampleValue)}`);
        });

        if (queryParams.length > 0) {
          curl += `?${queryParams.join("&")}`;
        }
      }
    } catch (e) {
      // 参数解析失败，尝试提取可能的参数名
      console.error("cURL GET参数解析失败:", e);

      if (typeof api.params === "string" && api.params.trim()) {
        const paramMatches = api.params.match(/"([^"]+)":/g);
        if (paramMatches && paramMatches.length > 0) {
          const queryParams: string[] = [];

          paramMatches.forEach((match: string) => {
            const paramName = match.replace(/[":]|"/g, "");
            queryParams.push(`${paramName}=example${paramName}`);
          });

          if (queryParams.length > 0) {
            curl += `?${queryParams.join("&")}`;
          }
        }
      }
    }
  }

  return curl;
};

// 生成JavaScript请求示例
const generateJsExample = (api: any): string => {
  const baseUrl = window.location.origin;
  const url = `${baseUrl}${api.path}`;
  let js = "";

  // 添加异步函数
  js += `// 使用Fetch API调用接口\n`;
  js += `async function call${api.name.replace(/\s+/g, "")}API() {\n`;

  // 处理GET请求的查询参数
  let fullUrl = url;
  if (api.method === "GET" && api.params) {
    try {
      const params = JSON.parse(api.params);
      if (Object.keys(params).length > 0) {
        const queryParams: string[] = [];

        // 为每个参数创建示例值
        Object.entries(params).forEach(([key, value]: [string, any]) => {
          let exampleValue: string;

          // 根据参数类型生成示例值
          switch (value.type?.toLowerCase()) {
            case "number":
              exampleValue = "1";
              break;
            case "boolean":
              exampleValue = "true";
              break;
            case "date":
              exampleValue = "2023-01-01";
              break;
            default:
              exampleValue = `example${key}`;
          }

          queryParams.push(`${key}=${encodeURIComponent(exampleValue)}`);
        });

        if (queryParams.length > 0) {
          fullUrl += `?${queryParams.join("&")}`;
        }
      }
    } catch (e) {
      // 参数解析失败，尝试提取可能的参数名
      console.error("JS GET参数解析失败:", e);

      if (typeof api.params === "string" && api.params.trim()) {
        const paramMatches = api.params.match(/"([^"]+)":/g);
        if (paramMatches && paramMatches.length > 0) {
          const queryParams: string[] = [];

          paramMatches.forEach((match: string) => {
            const paramName = match.replace(/[":]|"/g, "");
            queryParams.push(`${paramName}=example${paramName}`);
          });

          if (queryParams.length > 0) {
            fullUrl += `?${queryParams.join("&")}`;
          }
        }
      }
    }
  }

  // 添加请求选项
  js += `  const options = {\n`;
  js += `    method: '${api.method}',\n`;

  if (api.method !== "GET") {
    js += `    headers: {\n`;
    js += `      'Content-Type': 'application/json'\n`;
    js += `    },\n`;
  }

  // 添加请求体
  if (api.method !== "GET" && api.params) {
    try {
      const params = JSON.parse(api.params);
      if (Object.keys(params).length > 0) {
        const exampleBody: Record<string, any> = {};

        // 为每个参数创建示例值
        Object.entries(params).forEach(([key, value]: [string, any]) => {
          // 根据参数类型生成示例值
          switch (value.type?.toLowerCase()) {
            case "number":
              exampleBody[key] = 1;
              break;
            case "boolean":
              exampleBody[key] = true;
              break;
            case "date":
              exampleBody[key] = "2023-01-01";
              break;
            case "array":
              exampleBody[key] = [1, 2, 3];
              break;
            case "object":
              exampleBody[key] = { key: "value" };
              break;
            default:
              exampleBody[key] = `示例${key}`;
          }
        });

        js += `    body: JSON.stringify(${JSON.stringify(
          exampleBody,
          null,
          2
        )})\n`;
      }
    } catch (e) {
      // 参数解析失败，添加一个基本的请求体示例
      console.error("JS示例参数解析失败:", e);

      // 尝试从原始参数字符串中提取可能的参数名
      let exampleBody: Record<string, any> = {};

      if (typeof api.params === "string" && api.params.trim()) {
        const paramMatches = api.params.match(/"([^"]+)":/g);
        if (paramMatches && paramMatches.length > 0) {
          paramMatches.forEach((match: string) => {
            const paramName = match.replace(/[":]|"/g, "");
            exampleBody[paramName] = `示例${paramName}`;
          });
        }
      }

      // 如果无法提取参数，使用空对象
      if (Object.keys(exampleBody).length === 0) {
        exampleBody = { param: "value" };
      }

      js += `    body: JSON.stringify(${JSON.stringify(
        exampleBody,
        null,
        2
      )})\n`;
    }
  }

  js += `  };\n\n`;

  // 添加请求和响应处理
  js += `  try {\n`;
  js += `    const response = await fetch('${fullUrl}', options);\n`;
  js += `    const data = await response.json();\n`;
  js += `    console.log('API响应:', data);\n`;
  js += `    return data;\n`;
  js += `  } catch (error) {\n`;
  js += `    console.error('API调用失败:', error);\n`;
  js += `    throw error;\n`;
  js += `  }\n`;
  js += `}\n`;

  return js;
};

// 生成返回示例
const generateResponseExample = (api: any): string => {
  // 根据API类型生成不同的返回示例
  const crudType = api.crud_operation || "";
  const exampleResponse: Record<string, any> = {
    success: true,
  };

  switch (crudType) {
    case "READ_ALL":
      exampleResponse.data = [];
      // 添加示例数据项
      if (tableFields.value && tableFields.value.length > 0) {
        const exampleItem: Record<string, any> = {};
        tableFields.value.forEach((field) => {
          // 根据字段类型生成示例值
          switch (field.type.toLowerCase()) {
            case "int":
            case "integer":
            case "smallint":
            case "tinyint":
            case "mediumint":
            case "bigint":
              exampleItem[field.name] = 1;
              break;
            case "float":
            case "double":
            case "decimal":
              exampleItem[field.name] = 1.0;
              break;
            case "bool":
            case "boolean":
              exampleItem[field.name] = true;
              break;
            case "date":
              exampleItem[field.name] = "2023-01-01";
              break;
            case "datetime":
            case "timestamp":
              exampleItem[field.name] = "2023-01-01 12:00:00";
              break;
            default:
              exampleItem[field.name] = `示例${field.name}`;
          }
        });
        exampleResponse.data.push(exampleItem);
        // 添加第二个示例项
        exampleResponse.data.push({ ...exampleItem, id: 2 });
      } else {
        exampleResponse.data.push({ id: 1, name: "示例数据1" });
        exampleResponse.data.push({ id: 2, name: "示例数据2" });
      }
      exampleResponse.total = 2;
      break;

    case "READ_ONE":
      if (tableFields.value && tableFields.value.length > 0) {
        const exampleItem: Record<string, any> = {};
        tableFields.value.forEach((field) => {
          // 根据字段类型生成示例值
          switch (field.type.toLowerCase()) {
            case "int":
            case "integer":
            case "smallint":
            case "tinyint":
            case "mediumint":
            case "bigint":
              exampleItem[field.name] = 1;
              break;
            case "float":
            case "double":
            case "decimal":
              exampleItem[field.name] = 1.0;
              break;
            case "bool":
            case "boolean":
              exampleItem[field.name] = true;
              break;
            case "date":
              exampleItem[field.name] = "2023-01-01";
              break;
            case "datetime":
            case "timestamp":
              exampleItem[field.name] = "2023-01-01 12:00:00";
              break;
            default:
              exampleItem[field.name] = `示例${field.name}`;
          }
        });
        exampleResponse.data = exampleItem;
      } else {
        exampleResponse.data = { id: 1, name: "示例数据" };
      }
      break;

    case "CREATE":
      exampleResponse.message = "创建成功";
      exampleResponse.id = 1;
      break;

    case "UPDATE":
      exampleResponse.message = "更新成功";
      exampleResponse.affected = 1;
      break;

    case "DELETE":
      exampleResponse.message = "删除成功";
      exampleResponse.affected = 1;
      break;

    case "SEARCH":
      exampleResponse.data = [];
      // 添加示例搜索结果
      if (tableFields.value && tableFields.value.length > 0) {
        const exampleItem: Record<string, any> = {};
        tableFields.value.forEach((field) => {
          // 根据字段类型生成示例值
          switch (field.type.toLowerCase()) {
            case "int":
            case "integer":
            case "smallint":
            case "tinyint":
            case "mediumint":
            case "bigint":
              exampleItem[field.name] = 1;
              break;
            case "float":
            case "double":
            case "decimal":
              exampleItem[field.name] = 1.0;
              break;
            case "bool":
            case "boolean":
              exampleItem[field.name] = true;
              break;
            case "date":
              exampleItem[field.name] = "2023-01-01";
              break;
            case "datetime":
            case "timestamp":
              exampleItem[field.name] = "2023-01-01 12:00:00";
              break;
            default:
              exampleItem[field.name] = `示例${field.name}`;
          }
        });
        exampleResponse.data.push(exampleItem);
      } else {
        exampleResponse.data.push({ id: 1, name: "搜索结果1" });
      }
      exampleResponse.total = 1;
      break;

    default:
      // 默认返回示例
      exampleResponse.data = { message: "操作成功" };
  }

  return JSON.stringify(exampleResponse, null, 2);
};

// 页面加载时获取表列表
onMounted(() => {
  loadTables();
});
</script>

<style scoped>
.table-list {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  padding: 20px;
}

.table-list-content {
  flex: 1;
  margin-top: 20px;
}

.table-list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.empty-state {
  margin: 40px 0;
  display: flex;
  justify-content: center;
}

.table-sql {
  margin: 20px 0;
}

.table-sql pre {
  background-color: #f8f9fa;
  padding: 15px;
  border-radius: 4px;
  border: 1px solid #ebeef5;
  white-space: pre-wrap;
  font-family: monospace;
  max-height: 200px;
  overflow-y: auto;
}

.table-fields {
  margin: 20px 0;
}

.api-actions {
  margin-bottom: 20px;
  display: flex;
  justify-content: flex-end;
}

.api-preview-content {
  padding: 20px;
}

.preview-actions {
  margin-bottom: 20px;
  display: flex;
  justify-content: flex-end;
}

.markdown-preview {
  max-height: 60vh;
  overflow-y: auto;
  padding: 20px;
  background-color: #f9f9f9;
  border-radius: 8px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
    Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
}

.markdown-preview h1 {
  font-size: 28px;
  margin-top: 24px;
  margin-bottom: 16px;
  font-weight: 600;
  border-bottom: 1px solid #eaecef;
  padding-bottom: 0.3em;
}

.markdown-preview h2 {
  font-size: 24px;
  margin-top: 24px;
  margin-bottom: 16px;
  font-weight: 600;
  border-bottom: 1px solid #eaecef;
  padding-bottom: 0.3em;
}

.markdown-preview h3 {
  font-size: 20px;
  margin-top: 24px;
  margin-bottom: 16px;
  font-weight: 600;
}

.markdown-preview h4 {
  font-size: 16px;
  margin-top: 24px;
  margin-bottom: 16px;
  font-weight: 600;
}

.markdown-preview p {
  margin-top: 0;
  margin-bottom: 16px;
  line-height: 1.6;
}

.markdown-preview code {
  font-family: SFMono-Regular, Consolas, Liberation Mono, Menlo, monospace;
  padding: 0.2em 0.4em;
  margin: 0;
  font-size: 85%;
  background-color: rgba(27, 31, 35, 0.05);
  border-radius: 3px;
}

.markdown-preview pre {
  background-color: #f6f8fa;
  border-radius: 6px;
  padding: 16px;
  overflow: auto;
  font-family: SFMono-Regular, Consolas, Liberation Mono, Menlo, monospace;
  font-size: 85%;
  line-height: 1.45;
  margin-top: 0;
  margin-bottom: 16px;
}

.markdown-preview pre code {
  background-color: transparent;
  padding: 0;
  margin: 0;
  font-size: 100%;
  word-break: normal;
  white-space: pre;
  overflow-wrap: normal;
}

.markdown-preview table {
  border-collapse: collapse;
  width: 100%;
  margin-top: 0;
  margin-bottom: 16px;
}

.markdown-preview table th,
.markdown-preview table td {
  padding: 6px 13px;
  border: 1px solid #dfe2e5;
}

.markdown-preview table th {
  font-weight: 600;
  background-color: #f6f8fa;
}

.markdown-preview table tr:nth-child(2n) {
  background-color: #f6f8fa;
}

.markdown-preview ul,
.markdown-preview ol {
  padding-left: 2em;
  margin-top: 0;
  margin-bottom: 16px;
}

.markdown-preview li {
  margin-top: 0.25em;
}

.markdown-preview strong {
  font-weight: 600;
}

.markdown-preview em {
  font-style: italic;
}

.markdown-preview a {
  color: #0366d6;
  text-decoration: none;
}

.markdown-preview a:hover {
  text-decoration: underline;
}
</style>
