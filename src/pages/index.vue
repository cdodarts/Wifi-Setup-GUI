<template>
  <v-container class="page" fluid>
    <div class="page__bg" aria-hidden="true"></div>

    <v-container class="page__content" max-width="1100">
      <header class="hero">
        <div class="hero__brand">
          <img :src="cdoLogo" alt="CDO Darts" class="hero__logo" />
          <div class="hero__product">
            <span class="hero__tag hero__tag--product">CDO Vertex</span>
            <span class="hero__tag hero__tag--muted">WiFi Setup</span>
          </div>
        </div>
        <div class="hero__copy">
          <h1 class="hero__title">Connect your CDO Vertex</h1>
          <p class="hero__subtitle">
            Choose your WiFi network, enter the password, and we will take care of the rest.
          </p>
        </div>
      </header>

      <v-card class="panel" elevation="10">
        <v-card-text>
          <div class="panel__grid">
            <div class="panel__left">
              <h2 class="panel__title">Select a network</h2>
              <p class="panel__hint">
                Scan for nearby networks and pick the one your phone or laptop is on.
              </p>

              <v-select
                v-model="selectedSsid"
                :items="networkOptions"
                :loading="isScanning"
                label="Available WiFi"
                item-title="label"
                item-value="ssid"
                variant="outlined"
                density="comfortable"
                class="panel__field"
                :menu-props="{ maxHeight: 320 }"
                :disabled="isSubmitting"
              />

              <div class="panel__actions">
                <v-btn
                  variant="text"
                  color="primary"
                  :loading="isScanning"
                  :disabled="isSubmitting"
                  @click="scanNetworks"
                >
                  Rescan networks
                </v-btn>
                <span class="panel__meta">{{ scanStatus }}</span>
              </div>
            </div>

            <div class="panel__right">
              <h2 class="panel__title">Enter password</h2>
              <p class="panel__hint">
                Your details stay on this device. We use them only to connect.
              </p>

              <v-text-field
                v-model="password"
                :type="showPassword ? 'text' : 'password'"
                label="WiFi password"
                variant="outlined"
                density="comfortable"
                class="panel__field"
                :disabled="isSubmitting || isOpenNetwork"
                :placeholder="isOpenNetwork ? 'Open network does not require a password' : ''"
                @keyup.enter="submitConnection"
              >
                <template #append-inner>
                  <v-btn
                    icon
                    variant="text"
                    :disabled="isOpenNetwork"
                    @click="showPassword = !showPassword"
                  >
                    <v-icon>{{ showPassword ? 'mdi-eye-off' : 'mdi-eye' }}</v-icon>
                  </v-btn>
                </template>
              </v-text-field>

              <div class="panel__cta">
                <v-btn
                  class="connect-btn"
                  size="large"
                  :loading="isSubmitting"
                  :disabled="!canSubmit"
                  @click="submitConnection"
                >
                  Connect to WiFi
                </v-btn>
              </div>
            </div>
          </div>
        </v-card-text>
      </v-card>

      <section class="info">
        <div class="info__card">
          <h3>After connecting</h3>
          <p>
            Join the same WiFi network your CDO system connects to, then visit
            <strong>http://cdo-autodarts.local:3180</strong> to finish setup.
          </p>
        </div>
        <div class="info__card info__card--outline">
          <h3>Need help?</h3>
          <p>
            Make sure your router is in range and broadcasting on 2.4 GHz if your
            network has separate bands.
          </p>
        </div>
      </section>
    </v-container>

    <v-dialog v-model="showConnecting" persistent width="420">
      <v-card class="dialog">
        <v-card-text class="dialog__body">
          <v-progress-circular
            indeterminate
            size="56"
            width="5"
            color="primary"
          />
          <div>
            <h3>Connecting to WiFi</h3>
            <p>Please wait while we connect to {{ selectedSsid || 'your network' }}.</p>
          </div>
        </v-card-text>
      </v-card>
    </v-dialog>

    <v-dialog v-model="showSuccess" width="520">
      <v-card class="dialog dialog--success">
        <v-card-text>
          <h3>Connected successfully</h3>
          <p class="dialog__lead">
            Your CDO system is now on <strong>{{ statusSsid || selectedSsid }}</strong>.
          </p>
          <p>
            Join the same WiFi network and open
            <strong>http://cdo-autodarts.local:3180</strong> to continue setup.
          </p>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn color="primary" variant="flat" @click="showSuccess = false">Done</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="showError" width="520">
      <v-card class="dialog dialog--error">
        <v-card-text>
          <h3>Failed to connect</h3>
          <p class="dialog__lead">{{ errorMessage }}</p>
          <p>Please check the password and try again.</p>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn variant="text" @click="showError = false">Close</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script setup>
  import { computed, onMounted, ref } from 'vue'
  import cdoLogo from '@/assets/cdo-logo-nobg.png'

  const SCAN_URL = '/api/wifi/scan'
  const CONNECT_URL = '/api/wifi'
  const STATUS_URL = '/api/wifi/status'
  const POLL_INTERVAL_MS = 2000
  const POLL_ATTEMPTS = 15

  const networks = ref([])
  const selectedSsid = ref('')
  const password = ref('')
  const showPassword = ref(false)
  const isScanning = ref(false)
  const isSubmitting = ref(false)
  const scanStatus = ref('')
  const showConnecting = ref(false)
  const showSuccess = ref(false)
  const showError = ref(false)
  const errorMessage = ref('')
  const statusSsid = ref('')

  const networkOptions = computed(() =>
    networks.value.map((network) => ({
      ...network,
      label: formatNetworkLabel(network),
    }))
  )

  const isOpenNetwork = computed(() => {
    const selected = networks.value.find((network) => network.ssid === selectedSsid.value)
    return selected?.security === 'open'
  })

  const canSubmit = computed(() => {
    if (!selectedSsid.value || isSubmitting.value) return false
    if (isOpenNetwork.value) return true
    return password.value.trim().length > 0
  })

  const scanNetworks = async () => {
    isScanning.value = true
    scanStatus.value = 'Scanning...'

    try {
      const response = await fetch(SCAN_URL)
      const data = await response.json()

      if (!response.ok || !data.ok) {
        throw new Error(data?.error || 'Unable to scan for networks.')
      }

      networks.value = data.networks || []
      scanStatus.value = networks.value.length
        ? `Found ${networks.value.length} networks`
        : 'No networks found'

      if (!selectedSsid.value && networks.value.length) {
        selectedSsid.value = networks.value[0].ssid
      }
    } catch (error) {
      scanStatus.value = 'Scan failed'
      showErrorDialog(error)
    } finally {
      isScanning.value = false
    }
  }

  const submitConnection = async () => {
    if (!canSubmit.value) return

    isSubmitting.value = true
    showConnecting.value = true
    showError.value = false
    showSuccess.value = false
    errorMessage.value = ''
    statusSsid.value = ''

    try {
      const response = await fetch(CONNECT_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ssid: selectedSsid.value,
          password: isOpenNetwork.value ? '' : password.value,
        }),
      })

      const data = await response.json()

      if (!response.ok || !data.ok) {
        throw new Error(data?.error || 'Connection request failed.')
      }

      await pollStatus()
    } catch (error) {
      showConnecting.value = false
      isSubmitting.value = false
      showErrorDialog(error)
    }
  }

  const pollStatus = async () => {
    let attempts = 0

    const poll = async (resolve, reject) => {
      attempts += 1
      try {
        const response = await fetch(STATUS_URL)
        const data = await response.json()

        if (!response.ok || !data.ok) {
          throw new Error(data?.error || 'Unable to check connection status.')
        }

        if (data.connected) {
          statusSsid.value = data.ssid
          showConnecting.value = false
          showSuccess.value = true
          isSubmitting.value = false
          resolve()
          return
        }

        if (attempts >= POLL_ATTEMPTS) {
          throw new Error('Timed out waiting for WiFi connection.')
        }

        setTimeout(() => poll(resolve, reject), POLL_INTERVAL_MS)
      } catch (error) {
        showConnecting.value = false
        isSubmitting.value = false
        showErrorDialog(error)
        reject(error)
      }
    }

    return new Promise((resolve, reject) => poll(resolve, reject))
  }

  const formatNetworkLabel = (network) => {
    return network.ssid
  }

  const signalBars = (dbm) => {
    if (dbm == null) return '----'
    if (dbm > -50) return '++++'
    if (dbm > -60) return '+++-'
    if (dbm > -70) return '++--'
    if (dbm > -80) return '+---'
    return '----'
  }

  const showErrorDialog = (error) => {
    errorMessage.value = error?.message || 'Failed to connect to WiFi.'
    showError.value = true
  }

  onMounted(() => {
    scanNetworks()
  })
</script>

<style scoped>
  :global(body) {
    font-family: 'Segoe UI', 'Tahoma', 'Verdana', 'Arial', sans-serif;
    background: #0f1115;
  }

  .page {
    min-height: 100vh;
    padding: 0;
    color: #f6f7f9;
    position: relative;
    overflow: hidden;
  }

  .page__bg {
    position: absolute;
    inset: 0;
    background:
      radial-gradient(circle at 20% 20%, rgba(242, 70, 40, 0.25), transparent 50%),
      radial-gradient(circle at 80% 10%, rgba(47, 120, 255, 0.2), transparent 45%),
      linear-gradient(120deg, #0f1115, #141924);
    z-index: 0;
  }

  .page__content {
    position: relative;
    z-index: 1;
    padding: 48px 24px 80px;
  }

  .hero {
    display: grid;
    gap: 20px;
    align-items: center;
    margin-bottom: 32px;
    text-align: center;
  }

  .hero__brand {
    display: grid;
    justify-items: center;
    gap: 12px;
  }

  .hero__logo {
    height: 120px;
    width: auto;
    filter: drop-shadow(0 10px 18px rgba(0, 0, 0, 0.35));
  }

  .hero__product {
    display: grid;
    gap: 6px;
    align-items: center;
    justify-items: center;
  }

  .hero__tag {
    font-size: 0.95rem;
    text-transform: uppercase;
    letter-spacing: 0.14em;
    color: rgba(246, 247, 249, 0.95);
  }

  .hero__tag--product {
    font-size: 1.9rem;
    letter-spacing: 0.18em;
  }

  .hero__tag--muted {
    color: rgba(246, 247, 249, 0.95);
  }

  .hero__title {
    font-size: clamp(2.2rem, 3vw, 3.4rem);
    margin: 0 0 8px;
  }

  .hero__subtitle {
    margin: 0 auto;
    font-size: 1.1rem;
    color: rgba(246, 247, 249, 0.9);
    max-width: 640px;
  }

  .panel {
    border-radius: 24px;
    background: rgba(16, 20, 28, 0.9);
    border: 1px solid rgba(255, 255, 255, 0.06);
    backdrop-filter: blur(18px);
  }

  .panel__grid {
    display: grid;
    gap: 32px;
    grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  }

  .panel__title {
    margin-bottom: 6px;
    color: rgba(246, 247, 249, 0.98);
  }

  .panel__hint {
    color: rgba(246, 247, 249, 0.95);
    margin-bottom: 20px;
  }

  .panel__field {
    margin-bottom: 12px;
  }

  :deep(.v-field--variant-outlined .v-field__outline__start),
  :deep(.v-field--variant-outlined .v-field__outline__end),
  :deep(.v-field--variant-outlined .v-field__outline__notch) {
    border-color: rgba(246, 247, 249, 0.65);
  }

  :deep(.v-field--variant-outlined.v-field--focused .v-field__outline__start),
  :deep(.v-field--variant-outlined.v-field--focused .v-field__outline__end),
  :deep(.v-field--variant-outlined.v-field--focused .v-field__outline__notch) {
    border-color: rgba(246, 247, 249, 0.95);
  }

  :deep(.v-field--variant-outlined) {
    --v-field-border-opacity: 1;
    --v-field-border-color: rgba(246, 247, 249, 0.85);
  }

  :deep(.v-field--variant-outlined.v-field--focused) {
    --v-field-border-color: rgba(246, 247, 249, 0.95);
  }

  :deep(.v-field--variant-outlined .v-field__outline) {
    color: rgba(246, 247, 249, 0.85);
    opacity: 1;
  }

  :deep(.v-field--variant-outlined.v-field--focused .v-field__outline) {
    color: rgba(246, 247, 249, 0.95);
  }

  :deep(.v-field__input) {
    color: rgba(246, 247, 249, 0.98);
  }

  :deep(.v-field-label) {
    color: rgba(246, 247, 249, 0.95);
  }

  :deep(.v-field__input::placeholder) {
    color: rgba(246, 247, 249, 0.85);
    opacity: 1;
  }

  :deep(.v-field__append-inner .v-icon),
  :deep(.v-field__append-inner .v-btn) {
    color: rgba(246, 247, 249, 0.95);
  }

  .panel__actions {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    color: rgba(246, 247, 249, 0.95);
  }

  .panel__meta {
    font-size: 0.85rem;
  }

  .panel__cta {
    display: flex;
    justify-content: flex-start;
  }

  .connect-btn {
    background: linear-gradient(135deg, #25b269, #1f8f58);
    color: #ffffff;
    font-weight: 600;
    text-transform: none;
    letter-spacing: 0.02em;
    padding-inline: 36px;
    box-shadow: 0 16px 30px rgba(24, 132, 74, 0.35);
  }

  .info {
    margin-top: 36px;
    display: grid;
    gap: 20px;
    grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  }

  .info__card {
    background: rgba(20, 24, 34, 0.8);
    border-radius: 18px;
    padding: 20px 22px;
    border: 1px solid rgba(255, 255, 255, 0.06);
    color: rgba(246, 247, 249, 0.95);
  }

  .info__card--outline {
    background: transparent;
    border: 1px solid rgba(255, 255, 255, 0.15);
  }

  .dialog {
    border-radius: 18px;
    background: #121620;
    color: rgba(246, 247, 249, 0.98);
  }

  .dialog__body {
    display: flex;
    gap: 16px;
    align-items: center;
  }

  .dialog p {
    color: rgba(246, 247, 249, 0.95);
  }

  .dialog__lead {
    margin-top: 6px;
    color: rgba(246, 247, 249, 0.95);
  }

  .dialog--success {
    border: 1px solid rgba(37, 178, 105, 0.4);
  }

  .dialog--error {
    border: 1px solid rgba(242, 70, 40, 0.35);
  }

  @media (max-width: 720px) {
    .page__content {
      padding-top: 32px;
    }

    .hero__logo {
      height: 120px;
    }
  }
</style>
