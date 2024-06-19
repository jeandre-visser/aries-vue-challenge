<template>
  <div class="container">
    <h1>Options Profit Calculator</h1>
    <p class="description">
      This Options Profit Calculator helps you analyze the risk and reward of
      your options strategies. Follow the steps below to get started:
    </p>
    <ol class="steps">
      <li>Enter the details of up to four options contracts.</li>
      <li>Click "Add Option" to include the contract in your analysis.</li>
      <li>
        View the risk & reward graph and summary of your options strategy.
      </li>
    </ol>
    <form @submit.prevent="addOption">
      <div class="form-group">
        <label for="type">Option Type:</label>
        <select v-model="newOption.type" required>
          <option value="call">Call</option>
          <option value="put">Put</option>
        </select>
      </div>
      <div class="form-group">
        <label for="strike">Strike Price:</label>
        <input type="number" v-model="newOption.strike" required />
      </div>
      <div class="form-group">
        <label for="premium">Premium:</label>
        <input type="number" v-model="newOption.premium" required />
      </div>
      <button type="submit" class="add-button" :disabled="isAddButtonDisabled">
        {{ editIndex === -1 ? "Add Option" : "Update Option" }}
      </button>
    </form>
    <div id="chart"></div>
    <div v-if="hasCallOptions">
      <div>
        <h2>Summary</h2>
        <p class="profit">Max Profit: {{ maxProfit }}</p>
        <p class="loss">Max Loss: {{ maxLoss }}</p>
        <p>Break Even Points:</p>
        <ul class="break-even-points">
          <li v-for="(point, index) in displayedBreakEvenPoints" :key="index">
            {{ point }}
          </li>
        </ul>
        <button v-if="breakEvenPoints.length > 60" @click="toggleShowAll">
          {{ showAll ? "Hide" : "Expand" }}
        </button>
      </div>
    </div>
    <div v-if="optionsData.length > 0">
      <h2>Options List</h2>
      <ul>
        <li v-for="(option, index) in optionsData" :key="index">
          {{ option.type }} - Strike: {{ option.strike }} - Premium:
          {{ option.premium }}
          <button class="edit" @click="editOption(index)">Edit</button>
          <button @click="deleteOption(index)">Delete</button>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";

export default {
  name: "CodingChallenge",
  data() {
    return {
      optionsData: [],
      newOption: {
        type: "call",
        strike: "",
        premium: "",
      },
      editIndex: -1,
      maxProfit: 0,
      maxLoss: 0,
      breakEvenPoints: [],
      showAll: false,
    };
  },
  computed: {
    hasCallOptions() {
      return this.optionsData.some((option) => option.type === "call");
    },
    displayedBreakEvenPoints() {
      return this.showAll
        ? this.breakEvenPoints
        : this.breakEvenPoints.slice(0, 60);
    },
    isAddButtonDisabled() {
      return this.newOption.strike === "" || this.newOption.premium === "";
    },
  },
  watch: {
    optionsData: {
      handler() {
        this.calculateRiskReward();
        if (this.hasCallOptions) {
          this.drawChart();
        } else {
          d3.select("#chart").selectAll("*").remove(); // Clear chart if no call options
        }
      },
      deep: true,
    },
  },
  methods: {
    addOption() {
      if (this.editIndex === -1) {
        if (this.optionsData.length < 4) {
          this.optionsData.push({ ...this.newOption });
        } else {
          alert("You can only add up to four options contracts.");
        }
      } else {
        this.optionsData.splice(this.editIndex, 1, { ...this.newOption });
        this.editIndex = -1;
      }
      this.resetForm();
      this.$nextTick(() => {
        this.calculateRiskReward();
        if (this.hasCallOptions) {
          this.drawChart();
        }
      });
    },
    editOption(index) {
      this.newOption = { ...this.optionsData[index] };
      this.editIndex = index;
    },
    deleteOption(index) {
      this.optionsData.splice(index, 1);
    },
    resetForm() {
      this.newOption = { type: "call", strike: "", premium: "" };
    },
    calculateRiskReward() {
      let maxProfit = -Infinity;
      let maxLoss = Infinity;
      let breakEvenPoints = [];

      for (let price = 0; price <= 200; price += 1) {
        let profitLoss = 0;

        this.optionsData.forEach((option) => {
          if (option.type === "call") {
            profitLoss += Math.max(0, price - option.strike) - option.premium;
          } else if (option.type === "put") {
            profitLoss += Math.max(0, option.strike - price) - option.premium;
          }
        });

        if (profitLoss > maxProfit) maxProfit = profitLoss;
        if (profitLoss < maxLoss) maxLoss = profitLoss;
        if (profitLoss === 0) breakEvenPoints.push(price);
      }

      this.maxProfit = maxProfit;
      this.maxLoss = maxLoss;
      this.breakEvenPoints = breakEvenPoints;
    },
    drawChart() {
      d3.select("#chart").selectAll("*").remove(); // Clear previous chart

      const margin = { top: 20, right: 30, bottom: 50, left: 50 };
      const width = 500 - margin.left - margin.right;
      const height = 350 - margin.top - margin.bottom;

      const data = [];
      for (let price = 0; price <= 200; price += 1) {
        let profitLoss = 0;
        this.optionsData.forEach((option) => {
          if (option.type === "call") {
            profitLoss += Math.max(0, price - option.strike) - option.premium;
          } else if (option.type === "put") {
            profitLoss += Math.max(0, option.strike - price) - option.premium;
          }
        });
        data.push({ price, profitLoss });
      }

      const svg = d3
        .select("#chart")
        .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", height + margin.top + margin.bottom)
        .append("g")
        .attr("transform", `translate(${margin.left},${margin.top})`);

      const x = d3.scaleLinear().domain([0, 200]).range([0, width]);
      const y = d3
        .scaleLinear()
        .domain([
          d3.min(data, (d) => d.profitLoss),
          d3.max(data, (d) => d.profitLoss),
        ])
        .range([height, 0]);

      const line = d3
        .line()
        .x((d) => x(d.price))
        .y((d) => y(d.profitLoss));

      svg
        .append("path")
        .datum(data)
        .attr("fill", "none")
        .attr("stroke", "steelblue")
        .attr("stroke-width", 1.5)
        .attr("d", line);

      svg
        .append("g")
        .attr("transform", `translate(0,${height})`)
        .call(d3.axisBottom(x));

      svg.append("g").call(d3.axisLeft(y));

      // X-axis label
      svg
        .append("text")
        .attr("text-anchor", "end")
        .attr("x", width / 2)
        .attr("y", height + margin.bottom - 10)
        .text("Price of Underlying at Expiry");

      // Y-axis label
      svg
        .append("text")
        .attr("text-anchor", "end")
        .attr("transform", "rotate(-90)")
        .attr("x", -height / 2)
        .attr("y", -margin.left + 15)
        .text("Profit/Loss");
    },
    toggleShowAll() {
      this.showAll = !this.showAll;
    },
  },
};
</script>

<style scoped>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 0 20px;
  font-family: Arial, sans-serif;
}

h1 {
  font-size: 24px;
  margin-bottom: 20px;
  text-align: center;
  padding-bottom: 10px;
  border-bottom: 1px solid #ccc;
}

.description {
  font-size: 14px;
  margin-bottom: 10px;
  color: #000;
  background-color: lightpink;
}

.steps {
  font-size: 14px;
  margin-bottom: 20px;
  padding-left: 20px;
  color: #555;
  list-style-type: ;
}

.steps li {
  margin-bottom: 5px;
  border: none;
  display: list-item;
  margin: 0 auto;
  text-align: left;
}

form {
  display: flex;
  flex-direction: column;
  margin-bottom: 20px;
  gap: 8px;
  padding: 12px;
  border-radius: 12px;
  border: 1px solid #ccc;
}

.form-group {
  margin-bottom: 10px;
  display: flex;
  justify-content: space-between;
  max-width: 340px;
  margin: 0 auto;
  width: 100%;
  align-items: center;
}

label {
  margin: 0 10px 10px 0;
  font-weight: bold;
  white-space: nowrap;
}

input,
select {
  padding: 8px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

button {
  padding: 10px;
  font-size: 16px;
  color: #fff;
  background-color: #007bff;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.add-button {
  margin: 10px auto;
  width: 100%;
}

.add-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

button:hover {
  background-color: #0056b3;
}

#chart {
  margin: 20px auto;
  width: 100%;
}

h2 {
  font-size: 20px;
  margin-top: 10px;
}

p {
  font-size: 16px;
  padding: 0.5rem 1rem;
  border-radius: 1rem;
  background-color: lightblue;
}

.profit {
  background-color: #d4edda;
}

.loss {
  background-color: #f8d7da;
}

ul {
  list-style-type: none;
  padding: 0;
}

ul.break-even-points {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 20px;
}

.break-even-points li {
  width: fit-content;
  border: 2px solid lightblue;
}

li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-bottom: 10px;
}

li.empty-list {
  justify-content: center;
  border-color: orange;
}

li button {
  padding: 5px 10px;
  font-size: 14px;
  background-color: #dc3545;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

li button:hover {
  background-color: #c82333;
}

li button.edit {
  background-color: #28a745;
  margin-right: 4px;
}

li button.edit:hover {
  background-color: #006400;
}
</style>
