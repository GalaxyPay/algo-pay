<template>
  <v-app>
    <v-app-bar app>
      <span
        class="text-vuet text-h5 ml-3 pt-2"
        style="cursor: pointer"
        @click="router.push('/')"
      >
        <algo-icon color="currentColor" :width="30" />lgoPay
      </span>
      <v-spacer />
      <v-btn
        @click="showSettings = true"
        icon="mdi-cog"
        :color="!addr ? 'red' : ''"
      />
    </v-app-bar>
    <v-main>
      <v-form ref="request" @submit.prevent="generateQr">
        <v-container v-show="!listen">
          <v-row>
            <v-col>
              <v-text-field
                v-model.number="currencyAmount"
                type="number"
                label="Amount"
                :rules="[required]"
                :prepend-inner-icon="currencyIcon"
                variant="outlined"
              />
            </v-col>
            <v-col>
              <v-select
                v-model="assetId"
                label="Paid In"
                :items="assets"
                item-title="name"
                item-value="id"
                variant="outlined"
              />
            </v-col>
            <v-col cols="12">
              <v-text-field
                v-model="userNote"
                label="Note (optional)"
                variant="outlined"
              />
            </v-col>
          </v-row>
          <v-row>
            <v-col class="text-center">
              <v-btn
                color="primary"
                variant="outlined"
                type="submit"
                :disabled="!addr"
              >
                Request Payment
              </v-btn>
            </v-col>
          </v-row>
        </v-container>
      </v-form>
      <v-container>
        <v-row v-if="listen">
          <v-col cols="12" class="text-center">Awaiting Payment...</v-col>
          <v-col cols="12" class="text-center">
            <VueQrcode
              :value="paymentCode"
              :color="{ dark: '#000', light: '#fff' }"
              type="image/png"
              :width="300"
            />
          </v-col>
          <v-col cols="12" class="text-center">
            <v-btn color="primary" variant="outlined" @click="listen = false">
              Cancel
            </v-btn>
          </v-col>
        </v-row>
      </v-container>
      <v-dialog v-model="showSettings" max-width="750" persistent>
        <v-card>
          <v-form ref="settings" @submit.prevent="saveSettings">
            <v-card-title>
              <v-icon class="mr-1">mdi-cog</v-icon>Settings</v-card-title
            >
            <v-container>
              <v-combobox
                v-model="address"
                :items="nfds"
                item-title="name"
                item-value="name"
                :return-object="false"
                label="Receiver Address or NFD"
                variant="outlined"
                spellcheck="false"
                @keyup="lookupNfd"
                :rules="[required, validAddress]"
              />
              <v-select
                label="Currency"
                v-model="currency"
                :items="currencies"
                variant="outlined"
              />
            </v-container>
            <v-card-actions>
              <v-spacer />
              <v-btn color="primary" variant="outlined" type="submit">
                Save
              </v-btn>
            </v-card-actions>
          </v-form>
        </v-card>
      </v-dialog>
    </v-main>
    <v-footer app>
      <v-row>
        Powered By&nbsp;
        <a href="https://vestige.fi" target="_blank">
          <Vestige :width="100" />
        </a>
        <v-spacer />
        Version {{ appVersion }}
      </v-row>
    </v-footer>
    <v-snackbar
      v-model="snackbar.display"
      :timeout="snackbar.timeout"
      :color="snackbar.color"
    >
      {{ snackbar.text }}
      <template #actions>
        <v-btn variant="text" @click="snackbar.display = false"> Close </v-btn>
      </template>
    </v-snackbar>
  </v-app>
</template>

<script setup>
import { ref, computed } from "vue";
import VueQrcode from "vue-qrcode";
import algosdk from "algosdk";
import { get, set } from "idb-keyval";
import Vestige from "@/components/svg/Vestige";
import AlgoIcon from "@/components/svg/AlgoIcon";

const appVersion = ref(__APP_VERSION__);
const baseUrl = "https://free-api.vestige.fi";
const algodUrl = "https://mainnet-api.algonode.cloud";
const algod = new algosdk.Algodv2("", algodUrl, "");

const showSettings = ref(false);
const address = ref();
const addr = ref();
const nfds = ref();
const prices = ref([]);
const currencies = computed(() => Object.keys(prices.value));
const currency = ref("USD");

const pricing = ref();
const note = ref();
const listen = ref(false);
const snackbar = ref({});

const settings = ref();
const request = ref();
const required = (v) => !!v || "Required";
const validAddress = (v) =>
  algosdk.isValidAddress(v) || /\.algo$/.test(v) || "Invalid Address";

const currencyIcon = computed(
  () => `mdi-currency-${currency.value.toLowerCase()}`
);
const currencyAmount = ref();
const userNote = ref();
const assets = ref();
const assetId = ref();

const asset = computed(() => assets.value.find((a) => a.id === assetId.value));
const amount = computed(() =>
  Math.round(
    (currencyAmount.value / pricing.value?.[currency.value]) *
      Math.pow(10, asset.value?.decimals)
  )
);
const paymentCode = computed(() => {
  let code = `algorand://${addr.value}?amount=${amount.value}&note=${note.value}`;
  if (assetId.value) code += `&asset=${assetId.value}`;
  return code;
});

async function fetchAsync(url) {
  const response = await fetch(url);
  const data = await response.json();
  return data;
}

async function setSnackbar(text, color = "info", timeout = 4000) {
  if (color == "error") timeout = 15000;
  snackbar.value = {
    text: text,
    color: color,
    timeout: timeout,
    display: true,
  };
}

async function lookupNfd() {
  nfds.value = address.value
    ? (
        await fetchAsync(
          `https://api.nf.domains/nfd/browse?name=${address.value}`
        )
      ).filter((nfd) => nfd.depositAccount)
    : [];
}

async function resolveAddress(address) {
  if (algosdk.isValidAddress(address)) {
    addr.value = address;
  } else {
    addr.value = (
      await fetchAsync(`https://api.nf.domains/nfd/${address}`)
    ).depositAccount;
  }
}

async function saveSettings() {
  const { valid } = await settings.value.validate();
  if (valid) {
    await resolveAddress(address.value);
    set("address", address.value);
    set("currency", currency.value);
    showSettings.value = false;
    const acctInfo = await algod.accountInformation(addr.value).do();
    const assetIds = acctInfo.assets.map((a) => a["asset-id"]);
    const algo = { id: 0, name: "Algo", decimals: 6 };
    assets.value = [algo];
    assetId.value = 0;
    await Promise.all(
      assetIds.map(async (id) => {
        const url = `${baseUrl}/asset/${id}`;
        const assetInfo = await fetchAsync(url);
        if (assetInfo) assets.value.push(assetInfo);
      })
    );
    set("assets", JSON.parse(JSON.stringify(assets.value)));
  }
}

async function getPricing(id) {
  pricing.value = id
    ? await fetchAsync(`${baseUrl}/asset/${id}/price`)
    : await fetchAsync(`${baseUrl}/currency/prices`);
}

async function generateQr() {
  const { valid } = await request.value.validate();
  if (valid) {
    await getPricing(assetId.value);
    const id = (Math.random() + 1).toString(36).substring(7);
    const noteObj = { id: id, cur: currency.value, amt: currencyAmount.value };
    if (userNote.value) noteObj.note = userNote.value;
    note.value = JSON.stringify(noteObj);
    const status = await algod.status().do();
    const round = status["last-round"];
    listen.value = true;
    console.log(paymentCode.value);
    let message;
    for (let i = 0; i < 100 && listen.value; i++) {
      if (i) await algod.statusAfterBlock(round + i).do();
      const { block } = await algod.block(round + i).do();
      if (
        block.txns?.some(
          (t) =>
            ((t.txn.type == "pay" &&
              t.txn.amt == amount.value &&
              algosdk.encodeAddress(t.txn.rcv) == addr.value) ||
              (t.txn.type == "axfer" &&
                t.txn.xaid == assetId.value &&
                t.txn.aamt == amount.value &&
                algosdk.encodeAddress(t.txn.arcv) == addr.value)) &&
            JSON.parse(Buffer.from(t.txn?.note, "base64").toString()).id == id
        )
      ) {
        message = "Payment Received";
        break;
      }
    }
    if (listen.value) {
      if (message) setSnackbar(message, "success");
      else setSnackbar("Timed Out");
    }
    listen.value = false;
  }
}

async function startup() {
  address.value = await get("address");
  currency.value = (await get("currency")) || "USD";
  assets.value = await get("assets");
  if (assets.value?.length) assetId.value = 0;
  prices.value = await fetchAsync(`${baseUrl}/currency/prices`);
  if (!address.value) {
    showSettings.value = true;
  } else {
    await resolveAddress(address.value);
  }
}
startup();
</script>
