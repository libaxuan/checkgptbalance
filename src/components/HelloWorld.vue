<template>
  <div>
    <h1><a href="https://iili.io/H4UPcoQ.jpg">OpenAI-SK信息查询</a></h1>
    <form>
      <textarea id="api-key-input" placeholder="请输入API KEY，每行一个"></textarea>
    </form>
    <div style="text-align: center;">
      <select id="api-url-select" v-model="selectedApiUrl" @change="handleApiUrlChange">
        <option value="https://openai.1rmb.tk">【社区proxy】 https://openai.1rmb.tk</option>
        <option value="https://api.askgptai.tech">【CloudFlare】 https://api.askgptai.tech</option>
        <option value="https://api.openai.com">【官方Magic】 https://api.openai.com</option>
        <option value="custom">自定义 ...</option>
      </select>
      <input type="text" id="custom-url-input" v-model="customApiUrl" placeholder="自定义URL"/>
      <button type="button" id="search-btn" @click="sendRequest()">查询</button>
      <button @click="sortTable('result-table', 3, 'desc')">剩余额度从大到小</button>
      <button @click="sortTable('result-table', 3, 'asc')">剩余额度从小到大</button>
      <button @click="exportTableData">导出所有数据</button>
      <button @click="exportValidData">导出有效数据</button>
    </div>
    <table id="result-table">
      <thead>
      <tr>
        <th>API KEY</th>
        <th style="width: 50px">总额度</th>
        <th style="width: 185px">已使用</th>
        <th style="width: 90px">剩余额度</th>
        <th style="width: 90px">截止日期</th>
        <th style="width: 120px">最高模型</th>
        <th style="width: 120px">是否绑卡</th>
      </tr>
      </thead>
      <tbody></tbody>
    </table>
    <footer>
      <a href="https://sensechat.vip" style="display: inline-block;">🍀免费AI问答🍀</a>
      <a href="https://autoaigpt.cn" style="display: inline-block;">👉GO HOME👈</a>
      <a href="https://itools.site" style="display: inline-block;">🍀免费万能工具🍀</a>
    </footer>

  </div>
</template>

<script>
export default {
  data() {
    return {
      selectedApiUrl: "https://openai.1rmb.tk", // 默认选择的API URL
      customApiUrl: "", // 自定义的API URL
      isEncrypted: false
    };
  },
  methods: {
    handleApiUrlChange() {
      let apiUrlSelect = document.getElementById("api-url-select");
      let customUrlInput = document.getElementById("custom-url-input");

      if (apiUrlSelect.value === "custom") {
        customUrlInput.style.display = "inline-block";
        customUrlInput.style.marginTop = "5px";
      } else {
        customUrlInput.style.display = "none";
      }
    },
    sendRequest() {
      let queriedApiKeys = [];

      async function checkBilling(apiKey, apiUrl) {
        const now = new Date();
        const startDate = new Date(now - 90 * 24 * 60 * 60 * 1000);
        const endDate = new Date(now.getTime() + 24 * 60 * 60 * 1000);
        const urlSubscription = `${apiUrl}/v1/dashboard/billing/subscription`; // 查是否订阅
        // eslint-disable-next-line no-unused-vars
        const urlBalance = `${apiUrl}/dashboard/billing/credit_grants`; // 查普通账单
        const urlUsage = `${apiUrl}/v1/dashboard/billing/usage?start_date=${formatDate(startDate)}&end_date=${formatDate(endDate)}`; // 查使用量
        const headers = {
          "Authorization": "Bearer " + apiKey,
          "Content-Type": "application/json"
        };
        try {
          let response = await fetch(urlSubscription, {headers});
          if (!response.ok) {
            console.log("您的账户已被封禁，请登录OpenAI进行查看。");
            return;
          }
          const subscriptionData = await response.json();
          const totalAmount = subscriptionData.hard_limit_usd;
          response = await fetch(urlUsage, {headers});
          const usageData = await response.json();
          const totalUsage = usageData.total_usage / 100;
          const remaining = totalAmount - totalUsage;
          const lastDate = new Date(subscriptionData.access_until * 1000);
          const year = lastDate.getFullYear();
          const month = lastDate.getMonth() + 1;
          const day = lastDate.getDate();
          const endDateString = year + '-' + month + '-' + day;
          const hasPaymentMethod = subscriptionData.has_payment_method ? 'Yes' : 'No';
          console.log(`Total Amount: ${totalAmount.toFixed(2)}`);
          console.log(`Used: ${totalUsage.toFixed(2)}`);
          console.log(`Remaining: ${remaining.toFixed(2)}`);
          console.log(`End Date: ${endDateString}`);
          const modelUrl = `${apiUrl}/v1/models`;
          response = await fetch(modelUrl, {headers});
          const data = await response.json();
          let gptModels = data.data.filter(model => model.id.includes("gpt"));
          let highestGPTModel = gptModels.reduce((prev, current) => {
            let prevVersion = parseFloat(prev.id.split("-")[1]);
            let currentVersion = parseFloat(current.id.split("-")[1]);
            return (currentVersion > prevVersion) ? current : prev;
          });
          return [totalAmount, totalUsage, remaining, endDateString, highestGPTModel.id, hasPaymentMethod];
        } catch (error) {
          console.error(error);
          return [null, null, null, null, null];
        }
      }

      function formatDate(date) {
        const year = date.getFullYear();
        const month = (date.getMonth() + 1).toString().padStart(2, '0');
        const day = date.getDate().toString().padStart(2, '0');

        return `${year}-${month}-${day}`;
      }

      // send request logic
      let apiKeyInput = document.getElementById("api-key-input");
      let apiUrlSelect = document.getElementById("api-url-select");
      let customUrlInput = document.getElementById("custom-url-input");

      if (apiKeyInput.value.trim() === "") {
        alert("请填写API KEY");
        return;
      }
      document.getElementById("result-table").getElementsByTagName('tbody')[0].innerHTML = "";
      let apiUrl = "";
      if (apiUrlSelect.value === "custom") {
        if (customUrlInput.value.trim() === "") {
          alert("请设置API链接");
          return;
        } else {
          apiUrl = this.customApiUrl; // 使用自定义的API URL
        }
      } else {
        apiUrl = this.selectedApiUrl; // 使用选择的API URL
      }
      let apiKeys = apiKeyInput.value.trim().split("\n");

      let tableBody = document.querySelector("#result-table tbody");
      for (let i = 0; i < apiKeys.length; i++) {
        let apiKey = apiKeys[i].trim();
        // 判断是否已经查询过
        if (queriedApiKeys.includes(apiKey)) {
          console.log(`API KEY ${apiKey} 已查询过，跳过此次查询`);
          continue;
        }
        queriedApiKeys.push(apiKey);
        checkBilling(apiKey, apiUrl).then((data) => {
          let row = document.createElement("tr");
          let apiKeyCell = document.createElement("td");
          apiKeyCell.textContent = apiKey;
          row.appendChild(apiKeyCell);
          if (data[0] === null) {
            let errorMessageCell = document.createElement("td");
            errorMessageCell.colSpan = "6";
            errorMessageCell.classList.add("status-error");
            errorMessageCell.textContent = "API请求失败，请检查其有效性或网络情况";
            row.appendChild(errorMessageCell);
          } else {
            let totalGrantedCell = document.createElement("td");
            totalGrantedCell.textContent = data[0].toFixed(2);
            row.appendChild(totalGrantedCell);
            let totalUsedCell = document.createElement("td");
            totalUsedCell.textContent = data[1].toFixed(2);
            row.appendChild(totalUsedCell);
            // TOTAL AVAILABLE
            let totalAvailableCell = document.createElement("td");
            totalAvailableCell.textContent = data[2].toFixed(2);
            row.appendChild(totalAvailableCell);
            let endDateCell = document.createElement("td");
            endDateCell.textContent = data[3];
            row.appendChild(endDateCell);
            let highestGPTModel = document.createElement("td");
            highestGPTModel.textContent = data[4];
            row.appendChild(highestGPTModel);
            let hasPaymentMethod = document.createElement("td");
            hasPaymentMethod.textContent = data[5];
            row.appendChild(hasPaymentMethod);
          }
          tableBody.appendChild(row);
          if (i === apiKeys.length - 1) {
            queriedApiKeys = [];
          }
        }).catch((error) => {
          console.error(error);
          let row = document.createElement("tr");
          let apiKeyCell = document.createElement("td");
          apiKeyCell.textContent = apiKey;
          row.appendChild(apiKeyCell);
          let errorMessageCell = document.createElement("td");
          errorMessageCell.colSpan = "6";
          errorMessageCell.style.width = "90px";
          errorMessageCell.classList.add("status-error");
          errorMessageCell.textContent =
              "API-KEY不正确或已失效";
          row.appendChild(errorMessageCell);
          tableBody.appendChild(row);
          if (i === apiKeys.length - 1) {
            queriedApiKeys = [];
          }
        });
      }
    },
    exportTableData() {
      let table = document.getElementById("result-table");
      let rows = table.getElementsByTagName("tr");
      let csvContent = "data:text/csv;charset=utf-8,";

      // 添加表头
      csvContent += "API KEY,总额度,已使用,剩余额度,截止日期,最高模型,是否绑卡\r\n";

      for (let i = 0; i < rows.length; i++) {
        let rowData = [];
        let cells = rows[i].getElementsByTagName("td");

        for (let j = 0; j < cells.length; j++) {
          rowData.push(cells[j].textContent);
        }

        let row = rowData.join(",");
        csvContent += row + "\r\n";
      }

      let encodedUri = encodeURI(csvContent);
      let link = document.createElement("a");
      link.setAttribute("href", encodedUri);
      link.setAttribute("download", "table_data.csv");
      document.body.appendChild(link);
      link.click();
    },
    // 导出有效key
    exportValidData() {
      let table = document.getElementById("result-table");
      let rows = table.getElementsByTagName("tr");
      let validRows = [];
      let csvContent = "data:text/csv;charset=utf-8,";

      // 添加表头
      csvContent += "API KEY,总额度,已使用,剩余额度,截止日期,最高模型,是否绑卡\r\n";

      let hasValidData = false; // 标记是否存在有效数据

      for (let i = 0; i < rows.length; i++) {
        let rowData = [];
        let cells = rows[i].getElementsByTagName("td");

        // 检查 cells 数组是否为空
        if (cells.length > 0) {
          let isInvalid = cells[1].textContent.includes("API-KEY不正确或已失效");
          let isRequestFailed = cells[1].textContent.includes("API请求失败，请检查其有效性或网络情况");
          let isValid = !isInvalid && !isRequestFailed;

          if (isValid) {
            hasValidData = true;
            for (let j = 0; j < cells.length; j++) {
              rowData.push(cells[j].textContent);
            }

            validRows.push(rowData.join(","));
          }
        }
      }

      if (hasValidData) {
        csvContent += validRows.join("\r\n");

        let encodedUri = encodeURI(csvContent);
        let link = document.createElement("a");
        link.setAttribute("href", encodedUri);
        link.setAttribute("download", "valid_data.csv");
        document.body.appendChild(link);
        link.click();
      } else {
        alert("暂无有效数据导出");
      }
    }
    ,
    sortTable(tableId, column, order) {
      let table = document.getElementById(tableId);
      let tbody = table.getElementsByTagName('tbody')[0];
      let rows = tbody.getElementsByTagName('tr');
      let rowArray = Array.from(rows);

      // 获取每行的API-KEY和状态
      let getKeyAndStatus = (row) => {
        let key = row.getElementsByTagName('td')[0]?.textContent;
        let status = row.getElementsByTagName('td')[1]?.textContent;
        return {key, status};
      };

      // 将API-KEY不正确或已失效移到数组末尾
      rowArray.sort(function (a, b) {
        let aData = getKeyAndStatus(a);
        let bData = getKeyAndStatus(b);

        if (aData.status === "API-KEY不正确或已失效") {
          return 1;
        } else if (bData.status === "API-KEY不正确或已失效") {
          return -1;
        } else {
          return 0;
        }
      });

      rowArray.sort(function (a, b) {
        let aValue = a.getElementsByTagName('td')[column]?.textContent || '';
        let bValue = b.getElementsByTagName('td')[column]?.textContent || '';
        if (column === 3) {
          aValue = parseFloat(aValue);
          bValue = parseFloat(bValue);
        }
        if (order === 'asc') {
          return aValue - bValue;
        } else {
          return bValue - aValue;
        }
      });

      for (let i = 0; i < rowArray.length; i++) {
        tbody.appendChild(rowArray[i]);
      }
    },
  },

  mounted() {
    let apiUrlSelect = document.getElementById("api-url-select");
    apiUrlSelect.addEventListener("change", this.handleApiUrlChange);

  },
};
</script>

