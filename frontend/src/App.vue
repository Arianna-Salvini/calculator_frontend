<template>
  <div class="calculator">
    <!-- Display for numerator and denominator and result -->
    <div class="display" :class="{ error: outcome === 'KO', success: outcome === 'OK' }">
      <p>{{ numerator || "Enter your operation" }} {{ operator || "" }} {{ denominator || "" }}</p>
      <p v-if="isLoading" class="loading">Calculating...</p>
      <p class="result">{{ result || "" }}</p>
    </div>

    <!-- If you click on a numeric operator move from numeratore to denominator -->
    <div class="operator">
      <button @click="setOperation('+')" :disabled="isLoading">+</button>
      <button @click="setOperation('-')" :disabled="isLoading">-</button>
      <button @click="setOperation('*')" :disabled="isLoading">*</button>
      <button @click="setOperation('/')" :disabled="isLoading">/</button>
    </div>

    <!-- Numeric keybord and equal button -->
    <div class="numeric_keybord">
      <!-- Numeric keybord -->
      <div class="button">

        <!-- Numbers 1 to 3 -->
        <div class="row">
          <button @click="appendNumber(1)" :disabled="isLoading">1</button>
          <button @click="appendNumber(2)" :disabled="isLoading">2</button>
          <button @click="appendNumber(3)" :disabled="isLoading">3</button>
        </div>
        <!-- Numbers 4 to 6 -->
        <div class="row">
          <button @click="appendNumber(4)" :disabled="isLoading">4</button>
          <button @click="appendNumber(5)" :disabled="isLoading">5</button>
          <button @click="appendNumber(6)" :disabled="isLoading">6</button>
        </div>
        <!-- Numbers 7 to 9 -->
        <div class="row">
          <button @click="appendNumber(7)" :disabled="isLoading">7</button>
          <button @click="appendNumber(8)" :disabled="isLoading">8</button>
          <button @click="appendNumber(9)" :disabled="isLoading">9</button>
        </div>

        <!-- Numbers 0 and clear button -->
        <div class="row">
          <button @click="appendNumber(0)" :disabled="isLoading">0</button>

          <!-- C clear button -->
          <button class="clear" @click="clearInput" :disabled="isLoading">C</button>
        </div>

      </div>

      <!-- Equal button-->
      <div class="equal">
        <button @click="calculateResult" :disabled="isLoading || isCalculated">=</button>
      </div>

    </div>

  </div>

</template>

<script>

import axios from 'axios';

export default {
  data() {
    return {
      numerator: "",
      denominator: "",
      operator: "",
      result: "",
      outcome: "",
      currentField: "numerator", // Define active field 
      haveResult: false, // Resul calculate
      isCalculated: false, //track if resul is already calculated
      isLoading: false,  // loading state
      isAborted: false
    };
  },

  methods: {
    // Add number to current field
    appendNumber(number) {
      if (this.result === "Error: time exceeded") {
        this.clearInput();
      }


      // Reset after having a result
      if (this.haveResult) {
        // Clear input
        this.clearInput();
        this.haveResult = false;
      }

      if (this.result === "Error: incomplete input") {
        this.clearInput();
      }

      if (this.currentField === "numerator") {
        this.numerator += number;
      } else {
        this.denominator += number;
      }

      this.isCalculated = false;
    },

    //  Set selected operator for operation
    setOperation(operator) {
      this.operator = operator;
      // Pass to next field (denominator)
      this.currentField = "denominator";
    },

    //Set clear button; reset all field and restart
    clearInput() {
      this.numerator = "";
      this.operator = "";
      this.denominator = "";
      this.result = "";
      this.currentField = "numerator";
      this.isCalculated = false;
      this.isAborted = false;
    },

    //Calculating result RESULT; url built and axios call

    async calculateResult() {

      // Block futher operation if a calculation is alreadi in progress
      if (this.isLoading) return;

      // Check field
      if (!this.numerator || !this.denominator || !this.operator) {

        this.result = "Error: incomplete input";
        return;
      }

      const urlBase = "http://localhost:8081/api/calculator/";
      // let url = "";

      // Object to map operator fro API endpoint (could add here oter futer operator)
      const operatorMap = {
        "+": "sum",
        "-": "sub",
        "*": "multiply",
        "/": "div"
      };

      // 🚩 Questo potrebbe essere superfluo in questo caso...
      // if (!operatorMap[this.operator]) {
      //   this.result = "Error: invalid operator";
      //   return;
      // }

      // Bulid URL
      const url = `${urlBase}${operatorMap[this.operator]}?a=${this.numerator}&b=${this.denominator}`;

      // start loading
      this.isLoading = true;
      this.isCalculated = true;

      // create an AbortController instance
      const abortController = new AbortController();
      const signalRequest = abortController.signal; //signal for request

      // Set a timeout to abort request (after 2 sec)
      this.timeoutOp = setTimeout(() => {
        this.isAborted = true;
        abortController.abort(); // trigger the abort
        this.isLoading = false;
        this.result = "Error: time exceeded"
        this.outcome = "KO";
      }, 2500);


      try {
        // request API with await
        const response = await axios.get(url, signalRequest);

        // if request finish then clear timeout
        clearTimeout(this.timeoutOp)

        if (this.isAborted) return;

        if (response.data.outcome === "OK") {
          this.result = response.data.result;
          this.outcome = response.data.outcome;
        } else {
          this.result = response.data.error;
          this.outcome = response.data.outcome;
        }

      } catch (error) {

        // if errors occured than clear timeout
        clearTimeout(this.timeoutOp)

        if (this.isAborted) return;

        // Add AbortError check (timeout)
        if (error.name === 'AbortError') {
          return;

        } else if (error.response) {

          if (error.response.data) {

            if (error.response.data.outcome === "KO") {
              // console.error("Error", error.message);
              this.result = error.response.data.error;
              this.outcome = error.response.data.outcome;
              // console.log(this.result);
            }
          } else {

            this.result = "Server Error";
            this.outcome = "KO";

          }
        }
      } finally {

        // end loading
        if (this.isAborted) return;

        this.haveResult = true;
        this.isLoading = false;
      };

    }
  }
}
</script>

<style scoped>
/* Calculator */
.calculator {
  background-color: orange;
  border-radius: 15px;
  padding: 20px;
  font-family: Arial, Helvetica, sans-serif;
  box-shadow: 10px 15px 12px #000;

  display: flex;
  flex-direction: column;
}

.display {
  width: 100%;
  min-height: 110px;
  background-color: #ddd;
  color: #000;
  text-align: end;
  border-radius: 15px;
  margin: 5px 0 20px;
  padding: 10px;
  box-shadow: 3px 3px 5px inset #8e8e8e;

  p {
    font-size: 25px;
  }

}

.result {
  animation: fadeIn 2s ease-in;
}

@keyframes fadeIn {
  0% {
    opacity: 0;
  }

  50% {
    opacity: 0.5;
  }

  100% {
    opacity: 1;
  }

}

.loading {
  color: #585858;
  text-align: end;
  padding-left: 5px;
  animation: blinkText 2s infinite ease-in-out;
}

button:disabled {
  background-color: antiquewhite;
  color: #585858;
  cursor: not-allowed;
  pointer-events: none;
}

@keyframes blinkText {
  0% {
    opacity: 0;
  }

  50% {
    opacity: 1;
  }

  100% {
    opacity: 0;
  }
}

/* buttons */

.operator {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;

  >button {
    width: 55px;
    height: 55px;
    font-size: 25px;
    border: 1px solid #8e8e8e;
    border-radius: 50%;
    padding: 5px;
    margin: 5px;
    box-shadow: 1px 2px 3px;
    cursor: pointer;

  }
}

.operator button:hover {
  background-color: #cccccc;
  box-shadow: 1px 2px 3px inset #000;

}

/* numeric_keybord */

.numeric_keybord {
  display: flex;

  .buttons {
    display: flex;
    flex-direction: column;
    align-items: center;

    .row {
      display: flex;
      align-items: center;
    }
  }

  button {
    font-size: 20px;
    padding: 15px 20px;
    margin: 5px;
    border: 1px solid #8e8e8e;
    border-radius: 50%;
    box-shadow: 1px 2px 3px #000;

    cursor: pointer;
  }

  button:hover {
    background-color: #cccccc;
    box-shadow: 1px 2px 3px inset #000;

  }

  /* ClearButton */

  .clear {
    width: 120px;
    border-radius: 30px;
  }

  /* Equals Button */
  .equal {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;

    >button {
      width: 60px;
      height: 245px;
      font-size: 30px;
      border-radius: 30px;

      p {
        font-size: 18px;
        font-weight: bold;
      }
    }
  }

}
</style>
