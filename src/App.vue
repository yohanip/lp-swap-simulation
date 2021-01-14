<template>
  <div id="app">
    <div class="container">
      <h1 class="p-4 text-center">
        Liquidity Provider Simulation
        <br />
        <a href="http://eggde.com">EggDex.com</a>
      </h1>
      <div class="card">
        <div class="card-body">
          <div>
            <label class="form-label">Swap Fee %</label>
            <input
              type="number"
              class="form-control"
              v-model="feePct"
              :disabled="simulationStarted"
            />
          </div>

          <div class="mt-2">
            <label class="form-label">TT</label>
            <input
              type="number"
              class="form-control"
              v-model="major"
              :disabled="simulationStarted"
            />
          </div>

          <div class="mt-2">
            <label class="form-label">IDR</label>
            <input
              type="number"
              class="form-control"
              v-model="minor"
              :disabled="simulationStarted"
            />
          </div>

          <div class="mt-2">
            <label class="form-label">IDR TO TT Occurence (%)</label>
            <input
              type="number"
              class="form-control"
              v-model="buyChancePct"
              :disabled="simulationStarted"
            />
          </div>

          <div class="mt-2">
            <label class="form-label">MAX IDR / swap</label>
            <input
              type="number"
              class="form-control"
              v-model="startingFund_"
              :disabled="simulationStarted"
            />
          </div>
        </div>
      </div>

      <h4 class="text-center">Price: {{ price().dp(6).toFormat() }} IDR/TT</h4>

      <div class="card bg-danger text-light">
        <div class="card-body text-center">
          <div>
            <label class="form-label">Simulation Delay (msec)</label>
            <input
              type="number"
              class="form-control text-center"
              v-model.number="simulationDelay"
            />
          </div>
          <div class="mt-2">
            <button
              class="btn btn-primary"
              @click="startSimulation"
              v-if="!simulationStarted"
            >
              Start
            </button>
            <button
              class="btn btn-primary"
              @click="simulationStarted = false"
              v-else
            >
              Stop
            </button>
          </div>
        </div>
      </div>

      <div class="card mt-4">
        <div class="card-body">
          <div class="d-flex text-center">
            <div class="flex-grow-1 text-center bg-warning py-1">
              <h5 class="mb-0">TT</h5>
              <span class="mb-0 d-block">
                Collected Fee:
                <strong>{{ collectedFee.major.toFormat(6) }}</strong>
              </span>
              <span class="text-danger">
                Liquidity Balance:
                <strong>
                  {{ simulation.major.plus(collectedFee.major).toFormat(6) }}
                </strong>
              </span>
            </div>
            <div class="flex-grow-1 text-center bg-success py-1">
              <h5 class="mb-0">IDR</h5>
              <span class="mb-0 d-block">
                Collected Fee:
                <strong>{{ collectedFee.minor.toFormat(6) }}</strong>
              </span>
              <span class="text-danger">
                Liquidity Balance:
                <strong>
                  {{ simulation.minor.plus(collectedFee.minor).toFormat(6) }}
                </strong>
              </span>
            </div>
          </div>
          <div class="table-responsive">
            <table class="table">
              <thead>
                <tr>
                  <th>#</th>
                  <th>Action</th>
                  <th>Swap Price</th>
                  <th>TT Liquidity</th>
                  <th>IDR Liquidity</th>
                  <th>Liquidity Value (IDR)</th>
                  <th>New Price</th>
                </tr>
              </thead>

              <tbody>
                <tr v-for="item in rows" :key="item.n">
                  <td>{{ item.n }}</td>
                  <td>
                    <span class="text-nowrap" v-html="item.note"></span>
                  </td>
                  <td>{{ item.swapPrice.toFormat(6) }}</td>
                  <td>{{ item.major.toFormat(6) }}</td>
                  <td>{{ item.minor.toFormat(6) }}</td>
                  <td>{{ item.minor.times(2).toFormat(6) }}</td>
                  <td>{{ item.minor.div(item.major).toFormat(6) }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Vue, Component } from "vue-property-decorator";
import Big from "bignumber.js";

function delay(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function calcOutput(input: Big, inputReserve: Big, outputReserve: Big) {
  // o/i = Ro/(Ri + i) -> o = Ro.i/(Ri + i)
  return input.times(outputReserve).div(inputReserve.plus(input));
}

@Component({})
export default class App extends Vue {
  feePct = "0.2";
  major = "100000";
  minor = "6500000";
  tradeAmountPct = "1";
  simulationDelay = 1500;

  buyChancePct = "55";

  simulation = {
    major: new Big(0),
    minor: new Big(0),
    n: 0,
    TTVolume: new Big(0),
  };

  price() {
    const major = new Big(this.major);
    const minor = new Big(this.minor);

    if (major.gt(0)) {
      return minor.div(major);
    }

    return new Big(0);
  }

  rows: {
    n: number;
    note: string;
    swapPrice: Big;
    // kondisi setelah swap
    major: Big;
    minor: Big;
  }[] = [];

  collectedFee = {
    major: new Big(0),
    minor: new Big(0),
  };

  private startingFund_ = "5000";

  simulationStarted = false;

  startSimulation() {
    this.rows = [];
    this.simulationStarted = true;
    this.collectedFee = {
      major: new Big(0),
      minor: new Big(0),
    };

    const startingFund = new Big(this.startingFund_); // IDR

    this.$nextTick(async function () {
      this.simulation = {
        major: new Big(this.major),
        minor: new Big(this.minor),
        n: 0,
        TTVolume: new Big(0),
      };

      // buy->sell->buy->sell
      let buyRate = 0;
      let cycle = 0;

      while (this.simulationStarted) {
        let r, buy;
        ++cycle;

        if (window.crypto && window.crypto.getRandomValues) {
          const arr = new Uint8Array(1);
          window.crypto.getRandomValues(arr);
          r = arr[0];
          buy =
            r <
            Math.floor(255 * new Big(this.buyChancePct).div(100).toNumber());
        } else {
          r = Math.floor(Math.random() * 10);
          buy = r < 5;
        }

        if (buy) {
          ++buyRate;
          console.log(
            "buy rate (%)",
            new Big(buyRate).div(cycle).times(100).toFormat(2)
          );
        }

        let major = this.simulation.major.times(1);
        let minor = this.simulation.minor.times(1);
        let note = "";
        let output: Big;
        let swapPrice: Big;

        if (buy) {
          // IN IDR, OUT TT
          const amount = startingFund.times(Math.random());
          const fee = amount.times(this.feePct).div(100);

          this.collectedFee.minor = this.collectedFee.minor.plus(fee);

          const tradeAmount = amount.minus(fee);

          output = calcOutput(tradeAmount, minor, major);
          note = `<span class="text-success">SWAP ${amount.toFormat(
            6
          )} IDR to ${output.toFormat(6)} TT</span>`;
          swapPrice = amount.div(output);

          major = major.minus(output);
          minor = minor.plus(tradeAmount);
        } else {
          // IN TT, OUT ID
          const amount = startingFund.div(65).times(Math.random());
          const fee = amount.times(this.feePct).div(100);
          this.collectedFee.major = this.collectedFee.major.plus(fee);
          const tradeAmount = amount.minus(fee);

          output = calcOutput(tradeAmount, major, minor);
          note = `<span class="text-danger">SWAP ${amount.toFormat(
            6
          )} TT to ${output.toFormat(6)} IDR</span>`;
          swapPrice = output.div(amount);

          major = major.plus(tradeAmount);
          minor = minor.minus(output);
        }

        this.simulation.major = major;
        this.simulation.minor = minor;

        this.rows.unshift({
          n: ++this.simulation.n,
          note,
          swapPrice,
          major,
          minor,
        });

        this.rows.splice(70);

        await delay(this.simulationDelay);
      }
    });
  }
}
</script>