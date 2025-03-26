<script setup>
import { BleClient, numbersToDataView, numberToUUID } from '@capacitor-community/bluetooth-le';

const LIGHT_SERVICE = numberToUUID(0x00FF);
const LIGHT_LED_STATE_CHAR = numberToUUID(0xFF01);
const LIGHT_BRIGHTNESS_CHAR = numberToUUID(0xFF02);
const LIGHT_FADE_STATE_CHAR = numberToUUID(0xFF03);
const LIGHT_FADE_CHAR = numberToUUID(0xFF04);
const device = ref(null);

const onDisconnect = async (deviceId) => {
  await BleClient.connect(deviceId, (deviceId) => onDisconnect(deviceId));
}

const requestDevice = async () => {
  device.value = await BleClient.requestDevice({
    namePrefix: "elilight",
    services: [LIGHT_SERVICE],
  });
}

const luminosity = ref(255);
const ledOn = ref(0);
const fadeOn = ref(0);
const fadeTime = ref(255);

watch(device, async () => {
  await BleClient.connect(device.value.deviceId, (deviceId) => onDisconnect(deviceId));

  const ledState = await BleClient.read(device.value.deviceId, LIGHT_SERVICE, LIGHT_LED_STATE_CHAR);
  ledOn.value = ledState.getUint8(0);

  const brightnessState = await BleClient.read(device.value.deviceId, LIGHT_SERVICE, LIGHT_BRIGHTNESS_CHAR);
  luminosity.value = brightnessState.getUint8(0);

  const fadeState = await BleClient.read(device.value.deviceId, LIGHT_SERVICE, LIGHT_FADE_STATE_CHAR);
  fadeOn.value = fadeState.getUint8(0);

  const fadeTimeState = await BleClient.read(device.value.deviceId, LIGHT_SERVICE, LIGHT_FADE_CHAR);
  console.log("OKKK")
  console.log(fadeTimeState.getUint8(0))
  fadeTime.value = fadeTimeState.getUint8(0);
})
watch(ledOn, async (value) => {
  if (!value) {
    await BleClient.write(device.value.deviceId, LIGHT_SERVICE, LIGHT_FADE_STATE_CHAR, numbersToDataView([value]));
  }
  await BleClient.write(device.value.deviceId, LIGHT_SERVICE, LIGHT_LED_STATE_CHAR, numbersToDataView([value]));
});
watch(luminosity, async (value) => {
  await BleClient.write(device.value.deviceId, LIGHT_SERVICE, LIGHT_BRIGHTNESS_CHAR, numbersToDataView([value]));
});
watch(fadeOn, async (value) => {
  if (value) {
    ledOn.value = value;
  }
  await BleClient.write(device.value.deviceId, LIGHT_SERVICE, LIGHT_FADE_STATE_CHAR, numbersToDataView([value]));
});
watch(fadeTime, async (value) => {
  await BleClient.write(device.value.deviceId, LIGHT_SERVICE, LIGHT_FADE_CHAR, numbersToDataView([value]));
});

onMounted(async () => {
  await BleClient.initialize();
})
</script>
<template>
  <div class="h-screen flex flex-col" v-if="device">
    <!-- Ligne 1 (80% hauteur) -->
    <div class="h-[80%] flex">
      <!-- Colonne 1 (50%) -->
      <div class="w-1/2 p-4 bg-gray-100 flex flex-col items-center justify-center border-r-2 border-b-2 border-solid">
        <div class="mb-4 flex flex-row">
          <p>Stroboscope</p> <Icon class="mt-1" name="uil:asterisk" />
        </div>
        <UButton @click="fadeOn = !fadeOn" color="neutral">
          {{ fadeOn ? 'Arrêter' : 'Activer' }}
        </UButton>
        <USlider v-if="fadeOn" v-model="fadeTime" color="neutral" class="mt-4" :min="10" :max="255"/>
      </div>
      <div class="w-1/2 p-4 bg-gray-100 flex flex-col items-center justify-center border-b-2 border-solid">
        <div class="mb-4 flex flex-row">
          <p>Luminosité</p> <Icon class="mt-1" name="uil:lightbulb-alt" />
        </div>
        <USlider v-model="luminosity" color="neutral" orientation="vertical" class="h-48" :max="255" :min="0"/>
      </div>
    </div>

    <!-- Ligne 2 (20% hauteur) -->
    <div class="h-[20%] bg-gray-100 p-4 flex align-middle items-center justify-center">
      <UButton @click="ledOn = !ledOn" color="neutral">
        {{ ledOn ? 'Eteindre' : 'Allumer' }}
      </UButton>
    </div>
  </div>
  <div v-else>
    <div class="h-screen flex flex-col justify-center items-center">
      <p class="mb-4">Aucune Elilight connecté!</p>
      <UButton @click="requestDevice" color="neutral">
        Connecter une Elilight
      </UButton>
    </div>
  </div>
</template>
