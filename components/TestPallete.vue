<template>
    <div class="test-pallet-component">
        <h2 class="text-[30px] font-semibold mb-4 text-white">Test Pallet Interface</h2>

        <div v-if="!selectedAccount" class="mb-4">
            <p class="text-red-600">Please select an account first</p>
        </div>

        <div v-else class="mb-6">
            <div class="mb-2">
                <span class="font-medium">Selected Account:</span>
                <div class="text-sm mt-1">
                    {{ selectedAccount.meta.name }} ({{ shortenAddress(selectedAccount.address) }})
                </div>
            </div>

            <!-- Method selection for your specific pallet -->
            <div class="mb-4">
                <label class="block text-sm font-medium text-gray-700">Select Method</label>
                <select v-model="selectedMethod" class="mt-1 block w-full p-2 border border-gray-300 rounded-md">
                    <option disabled value="">Select a method</option>
                    <option v-for="method in palletMethods" :key="method.method" :value="method.method">
                        {{ method.method }}
                    </option>
                </select>
                <div v-if="selectedMethodObj && selectedMethodObj.documentation" class="mt-1 text-sm text-gray-500">
                    {{ selectedMethodObj.documentation }}
                </div>
            </div>

            <!-- Dynamic form fields based on the selected method -->
            <div class="mb-4" v-if="paramFields.length > 0">
                <label class="block text-sm font-medium text-gray-700">Parameters</label>
                <div v-for="(param, index) in paramFields" :key="index" class="mt-2">
                    <label class="block text-xs text-gray-500">{{ param }}</label>
                    <input v-model="paramValues[index]" type="text" :placeholder="`Parameter ${index + 1}`"
                        class="mt-1 block w-full p-2 border border-gray-300 rounded-md" />
                </div>
            </div>

            <button @click="submitMethod" :disabled="!canSubmit || submitting"
                class="mt-4 px-4 py-2 bg-green-600 text-white rounded-md hover:bg-green-700 disabled:bg-gray-400">
                {{ submitting ? 'Submitting...' : 'Submit' }}
            </button>
        </div>

        <!-- Display chain state from your pallet -->
        <div class="mt-8">
            <h3 class="text-lg font-medium mb-3">Pallet Storage</h3>

            <div class="mb-4">
                <label class="block text-sm font-medium text-gray-700">Select Storage Item</label>
                <select v-model="selectedStorage" class="mt-1 block w-full p-2 border border-gray-300 rounded-md">
                    <option disabled value="">Select a storage item</option>
                    <option v-for="item in storageItems" :key="item.method" :value="item.method">
                        {{ item.method }}
                    </option>
                </select>
            </div>

            <div class="mb-4" v-if="storageParamFields.length > 0">
                <label class="block text-sm font-medium text-gray-700">Query Parameters</label>
                <div v-for="(param, index) in storageParamFields" :key="index" class="mt-2">
                    <input v-model="storageParamValues[index]" type="text" :placeholder="`Parameter ${index + 1}`"
                        class="mt-1 block w-full p-2 border border-gray-300 rounded-md" />
                </div>
            </div>

            <button @click="queryStorage" :disabled="!selectedStorage || querying"
                class="mt-4 px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 disabled:bg-gray-400">
                {{ querying ? 'Querying...' : 'Query Storage' }}
            </button>

            <div v-if="queryResult" class="mt-4">
                <h4 class="font-medium mb-2">Result:</h4>
                <pre class="bg-gray-100 p-4 rounded-md overflow-auto max-h-60">{{ formatResult(queryResult) }}</pre>
            </div>
        </div>

        <div v-if="txHash" class="mt-4">
            <h3 class="font-medium text-lg mb-2">Transaction Submitted:</h3>
            <div class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded">
                <p>Block Hash: {{ txHash }}</p>
            </div>
        </div>

        <div v-if="txError" class="mt-4">
            <h3 class="font-medium text-lg mb-2">Error:</h3>
            <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded">
                <p>{{ txError }}</p>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
const { $polkadot } = useNuxtApp()

const props = defineProps({
    selectedAccount: {
        type: Object,
        default: null
    }
})

// Extrinsic-related
const palletMethods = ref([])
const selectedMethod = ref('')
const paramFields = ref([])
const paramValues = ref([])
const txHash = ref(null)
const txError = ref(null)
const submitting = ref(false)

// Storage-related
const storageItems = ref([])
const selectedStorage = ref('')
const storageParamFields = ref([])
const storageParamValues = ref([])
const queryResult = ref(null)
const querying = ref(false)

const canSubmit = computed(() => {
    return props.selectedAccount && selectedMethod.value && !submitting.value
})

const selectedMethodObj = computed(() => {
    return palletMethods.value.find(method => method.method === selectedMethod.value)
})

// Init data
onMounted(async () => {
    try {
        await $polkadot.connect()
        palletMethods.value = await $polkadot.getPalletExtrinsics('testPallet')
        storageItems.value = await $polkadot.getPalletStorage('testPallet')
    } catch (err) {
        console.error('Init error:', err)
        txError.value = err.message
    }
})

// Watch extrinsic method
watch(selectedMethod, async (method) => {
    if (!method) return
    resetParams()
    try {
        const meta = $polkadot.api.tx.testPallet[method].meta
        if (meta?.args) {
            paramFields.value = meta.args.map(arg => arg.name.toString())
            paramValues.value = Array(paramFields.value.length).fill('')
        }
    } catch (err) {
        console.error('Error analyzing extrinsic:', err)
    }
})

// Watch storage item
watch(selectedStorage, async (storage) => {
    if (!storage) return
    resetStorageParams()
    try {
        const metadata = $polkadot.api.query.testPallet[storage]
        if (metadata.meta?.type?.isMap) {
            const numParams = metadata.meta.type.asMap.keys.length
            storageParamFields.value = Array(numParams).fill('')
            storageParamValues.value = Array(numParams).fill('')
        }
    } catch (err) {
        console.error('Error analyzing storage:', err)
    }
})

// Methods
function resetParams() {
    paramFields.value = []
    paramValues.value = []
    txHash.value = null
    txError.value = null
}

function resetStorageParams() {
    storageParamFields.value = []
    storageParamValues.value = []
    queryResult.value = null
    txError.value = null
}

function shortenAddress(address) {
    return address ? `${address.slice(0, 6)}...${address.slice(-4)}` : ''
}

async function submitMethod() {
    if (!canSubmit.value) return
    submitting.value = true
    txHash.value = null
    txError.value = null
    try {
        const args = paramValues.value.filter(p => p !== '')
        const result = await $polkadot.callTestPallet(props.selectedAccount.address, selectedMethod.value, args)
        txHash.value = result.blockHash
        setTimeout(queryStorage, 2000)
    } catch (err) {
        console.error('Submit error:', err)
        txError.value = err.message
    } finally {
        submitting.value = false
    }
}

async function queryStorage() {
    if (!selectedStorage.value) return
    querying.value = true
    queryResult.value = null
    txError.value = null
    try {
        const params = storageParamValues.value.filter(p => p !== '')
        queryResult.value = await $polkadot.queryChainState('testPallet', selectedStorage.value, params)
    } catch (err) {
        console.error('Query error:', err)
        txError.value = err.message
    } finally {
        querying.value = false
    }
}

function formatResult(result) {
    try {
        return result?.toHuman ? JSON.stringify(result.toHuman(), null, 2) : JSON.stringify(result, null, 2)
    } catch (e) {
        return String(result)
    }
}
</script>