<template>
  <div>
    <h1>GPT-SK信息查询</h1>
    <form>
      <textarea id="api-key-input" placeholder="请输入API KEY，每行一个"></textarea>
    </form>
    <div style="text-align: center;">
      <select id="api-url-select">
        <option value="https://openai.1rmb.tk">【社区proxy】 https://openai.1rmb.tk</option>
        <option value="https://openai.1rmb.tk">【CloudFlare】 https://api.askgptai.tech</option>
        <option value="https://api.openai.com">【官方Magic】 https://api.openai.com</option>
        <option value="custom">自定义 ...</option>
      </select>
      <button type="button" id="search-btn" @click="sendRequest()">查询</button>
      <button @click="sortTable('result-table', 3, 'desc')">剩余额度从大到小</button>
      <button @click="sortTable('result-table', 3, 'asc')">剩余额度从小到大</button>
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
      <footer>
        <a href="https://autoaigpt.cn">👉GO HOME👈</a>
        <br>
        <a href="https://sensechat.vip">🍀体验免费万能助手🍀</a>
        <br>
        <a href="https://itools.site">🍀免费万能工具🍀</a>
      </footer>
    </footer>
  </div>
</template>

<script>
export default {
  methods: {
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
          apiUrl = customUrlInput.value.trim();
        }
      } else {
        apiUrl = apiUrlSelect.value;
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
              "不正确或已失效的API-KEY";
          row.appendChild(errorMessageCell);
          tableBody.appendChild(row);
          if (i === apiKeys.length - 1) {
            queriedApiKeys = [];
          }
        });
      }
    },
    sortTable(tableId, column, order) {
      let table = document.getElementById(tableId);
      let tbody = table.getElementsByTagName('tbody')[0];
      let rows = tbody.getElementsByTagName('tr');
      let rowArray = Array.from(rows);

      rowArray.sort(function (a, b) {
        let aValue = a.getElementsByTagName('td')[column].textContent;
        let bValue = b.getElementsByTagName('td')[column].textContent;
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
    }
  },

  mounted() {
    let apiUrlSelect = document.getElementById("api-url-select");
    let customUrlInput = document.getElementById("custom-url-input");
    apiUrlSelect.addEventListener("change", function () {
      if (apiUrlSelect.value === "custom") {
        customUrlInput.style.display = "inline-block";
        customUrlInput.style.marginTop = "5px";
      } else {
        customUrlInput.style.display = "none";
      }
    });
    },

};
</script>

