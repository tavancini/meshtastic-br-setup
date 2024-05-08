<template>
  <header>
    <h1>Meshtastic BR Setup</h1>
    <h3 class="header-warning">ESTA FERRAMENTA <u>NÃO</u> TRANSMITE E <u>NÃO</u> ARMAZENA QUALQUER INFORMAÇÃO PREENCHIDA.</h3>
  </header>

  <main>
    <!-- PASSO 1 -->
    <h2>Passo 1</h2>
    <p>Conecte o seu dispositivo na porta USB do seu computador, pressione o botão abaixo e selecione o seu dispositivo na lista.</p>
    <button type="button" @click="connect">Conectar</button>


    <fieldset :disabled="!canEdit">
      <!-- PASSO 2 -->
      <h2>Passo 2</h2>
      <p>Nome e código do seu dispositivo.</p>

      <input v-model="inputLongName" id="inputLongName" type="text" placeholder="Nome - Ex.: Dispositivo Teste"><br/>
      <input v-model="inputShortName" id="inputShortName" type="text" placeholder="Código - Ex.: dtst" maxlength="4">


      <!-- PASSO 3 -->
      <h2>Passo 3</h2>
      <p>Configurações da sua rede WiFi:</p>
      <input v-model="inputWifiSsid" type="text" placeholder="Nome da rede WiFi"><br/>
      <input v-model="inputWifiPsk" type="text" placeholder="Senha da rede WiFi">

      <!-- PASSO 4 -->
      <h2>Passo 4</h2>
      <p>Tipo de posicionamento do seu dispositivo:</p>
      <input type="radio" id="inputPositionDynamic" name="inputPositionFixed" value="0" checked v-model="inputPositionFixed"><label for="inputPositionDynamic"> Posicionamento dinâmico <i>(se o seu dispositivo possuir GPS)</i></label><br/>
      <input type="radio" id="inputPositionFixed" name="inputPositionFixed" value="1" v-model="inputPositionFixed"><label for="inputPositionFixed"> Posicionamento fixo <i>(se o seu dispositivo não possuir GPS e você queira definir uma posição fixa)</i></label><br>
      <div v-if="inputPositionFixed == '1'">
        <input v-model="inputPositionAltitude" id="inputPositionAltitude" type="text" placeholder="Altitude - Ex.: 760"><br/>
        <input v-model="inputPositionLatitude" id="inputPositionLatitude" type="text" placeholder="Latitude - Ex.: -23.545691"><br/>
        <input v-model="inputPositionLongitude" id="inputPositionLongitude" type="text" placeholder="Longitude - Ex.: -46.634031">
      </div>

      <!-- PASSO 5 -->
      <h2>Passo 5</h2>
      <p>Pressione o botão abaixo e aguarde o processo terminar:</p>
      <p class="status-message" v-html="statusMessage"></p>
      <button type="button" @click="initSetup">Salvar</button>

    </fieldset>
  </main>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import {
  Client,
  SerialConnection,
  Protobuf
} from '@meshtastic/js';

const client = new Client();
let connection : SerialConnection;

const deviceLora = ref(Protobuf.Config.Config_LoRaConfig);
const devicePosition = ref(Protobuf.Mesh.Position);
const deviceNetwork = ref(Protobuf.Config.Config_NetworkConfig);
const deviceBluetooth = ref(Protobuf.Config.Config_BluetoothConfig);
const deviceMqtt = ref(Protobuf.ModuleConfig.ModuleConfig);
const devicePrimaryChannel = ref(Protobuf.ModuleConfig.ModuleConfig);

const inputLongName = ref('');
const inputShortName = ref('');
const inputWifiSsid = ref('');
const inputWifiPsk = ref('');
const inputPositionAltitude = ref('');
const inputPositionLatitude = ref('');
const inputPositionLongitude = ref('');
const inputPositionFixed = ref('0');
const canEdit = ref(false);

const currentSetupStep = ref(0);
const totalSetupStep = 5;
const statusMessage = ref('');

const debug = async (title: string, data: any) => {
  console.log(title.toUpperCase() + ': ' + JSON.stringify(data, null, 4));
};

const updateStatus = async (message:string) => {
  console.log(message);
  statusMessage.value = message;
};

const connect = async () => {
  let port: SerialPort = null;

  try {
    port = await navigator.serial.requestPort();
  } catch(e) {
    return;
  }
  
  connection = client.createSerialConnection();
  await connection
    .connect({
      port,
      baudRate: 115200,
      concurrentLogOutput: true,
    })
    .catch((e: Error) => console.log(`Unable to Connect: ${e.message}`));
  canEdit.value = true;

  connection.events.onUserPacket.subscribe((userPacket) => {
    if (inputLongName.value.length == 0) inputLongName.value = userPacket.data.longName;
    if (inputShortName.value.length == 0) inputShortName.value = userPacket.data.shortName;
  });

  connection.events.onChannelPacket.subscribe((channel) => {
    if (channel.index == 0) {
      devicePrimaryChannel.value = channel;
    }
  });

  connection.events.onModuleConfigPacket.subscribe((moduleConfig) => {
    switch (moduleConfig.payloadVariant.case) {
      case 'mqtt':
        deviceMqtt.value = moduleConfig.payloadVariant.value;
        break;
    }
  });

  connection.events.onConfigPacket.subscribe((config) => {
    switch (config.payloadVariant.case) {
      case 'lora':
        deviceLora.value = config.payloadVariant.value;
        break;
      case 'bluetooth':
        deviceBluetooth.value = config.payloadVariant.value;
        break;
      case 'position':
        devicePosition.value = config.payloadVariant.value;
        if (currentSetupStep.value == 0) {
          inputPositionFixed.value = (devicePosition.value.fixedPosition) ? '1' : '0';
        }
        break;
      case 'network':
        deviceNetwork.value = config.payloadVariant.value;
        if (currentSetupStep.value == 0) {
          inputWifiSsid.value = deviceNetwork.value.wifiSsid;
          inputWifiPsk.value = deviceNetwork.value.wifiPsk;
        }
        break;
    }
  });
};

const initSetup = async () => {
  canEdit.value = false;
  // Owner
  updateStatus('Configurando o nome do dispositivo...');
  const userData = new Protobuf.Mesh.User({
      longName: inputLongName.value,
      shortName: inputShortName.value,
  });
  connection.setOwner(userData);

  connection.commitEditSettings();
  currentSetupStep.value = 1;

  connection.events.onDeviceStatus.subscribe((status) => {
    if (status == 7) {
      switch(currentSetupStep.value) {
        case 1: // LoRa region
        updateStatus('Configurando região LoRa do dispositivo...');
          const loraData = new Protobuf.Config.Config({
            payloadVariant: {
              case: 'lora',
              value: new Protobuf.Config.Config_LoRaConfig({
                ...deviceLora.value,
                region: Protobuf.Config.Config_LoRaConfig_RegionCode.ANZ,
              }),
            },
          });
          connection.setConfig(loraData);

          connection.commitEditSettings();
          currentSetupStep.value = currentSetupStep.value + 1;

        /////////////////////////////////////////////////////////////////////////
        case 2: // WiFi & Bluetooth
          updateStatus('Configurando bluetooth...');
          const bluetoothData = new Protobuf.Config.Config({
            payloadVariant: {
              case: 'bluetooth',
              value: new Protobuf.Config.Config_BluetoothConfig({
                ...deviceBluetooth.value,
                enabled: false,
              }),
            },
          });
          connection.setConfig(bluetoothData);

          updateStatus('Configurando WiFi...');
          const networkData = new Protobuf.Config.Config({
            payloadVariant: {
              case: 'network',
              value: new Protobuf.Config.Config_NetworkConfig({
                ...deviceNetwork.value,
                wifiEnabled: true,
                wifiPsk: inputWifiPsk.value,
                wifiSsid: inputWifiSsid.value,
              }),
            },
          });
          connection.setConfig(networkData);

          connection.commitEditSettings();
          currentSetupStep.value = currentSetupStep.value + 1;
          break;

        /////////////////////////////////////////////////////////////////////////
        case 3: // Position
          if (inputPositionFixed.value == '1') {
            updateStatus('Configurando a posição fixa...');
            const positionData = new Protobuf.Mesh.Position({
              altitude: parseInt(inputPositionAltitude.value),
              latitudeI: Math.round(parseFloat(inputPositionLatitude.value) / 1e-7),
              longitudeI: Math.round(parseFloat(inputPositionLongitude.value) / 1e-7),
            });
            connection.setPosition(positionData);
          }

          updateStatus('Configurando tipo de posicionamento...');
          const positionConfData = new Protobuf.Config.Config({
            payloadVariant: {
              case: 'position',
              value: new Protobuf.Config.Config_PositionConfig({
                ...devicePosition.value,
                fixedPosition: (inputPositionFixed.value == '1' ? true : false),
              }),
            },
          });
          connection.setConfig(positionConfData);

          connection.commitEditSettings();
          currentSetupStep.value = currentSetupStep.value + 1;
          break;

        /////////////////////////////////////////////////////////////////////////
        case 4: // Primary Channel & MQTT
          updateStatus('Configurando canal primário...');
          const primaryChannelData = new Protobuf.Channel.Channel({
            ...devicePrimaryChannel.value,
            settings: new Protobuf.Channel.ChannelSettings({
              ...devicePrimaryChannel.value.settings,
              uplinkEnabled: true,
              downlinkEnabled: true,
            }),
          });
          connection.setChannel(primaryChannelData);

          updateStatus('Configurando MQTT server...');
          const moduleConfigMqttData = new Protobuf.ModuleConfig.ModuleConfig({
            payloadVariant: {
              case: 'mqtt',
              value: new Protobuf.ModuleConfig.ModuleConfig_MQTTConfig({
                ...deviceMqtt.value,
                enabled: true,
                address: 'platform.meshbrasil.com:1883',
                username: 'meshdev',
                password: 'large4cats',
                root: 'meshdev',
                encryptionEnabled: true,
                tlsEnabled: false,
              }),
            },
          });
          connection.setModuleConfig(moduleConfigMqttData);

          connection.commitEditSettings();
          currentSetupStep.value = currentSetupStep.value + 1;
          break;
        case 5:
          finishSetup();
          break;
      }
    }
  });
};

const finishSetup = async() => {
  currentSetupStep.value = 0;
  canEdit.value = false;

  inputLongName.value = '';
  inputShortName.value = '';
  inputWifiSsid.value = '';
  inputWifiPsk.value = '';
  inputPositionAltitude.value = '';
  inputPositionLatitude.value = '';
  inputPositionLongitude.value = '';
  inputPositionFixed.value = '0';

  connection.disconnect();
  updateStatus('Pronto! Agora você pode fazer os últimos ajustes utilizando a interface oficial em <a href="https://client.meshtastic.org" target="_blank">https://client.meshtastic.org</a> !');
};
</script>
