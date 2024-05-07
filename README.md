# Meshtastic BR Setup

Interface web que facilita a configuração inicial de um dispositivo LoRa para ingressar à rede Meshtastic do Brasil.

## A ferramenta fará as seguintes configurações:
1. Nome e código do dispositivo.
2. Região: `ANZ`.
3. Bluetooth será desativado.
4. WiFi será ativado, junto com a definição do nome da rede e senha.
5. Posicionamento dinâmico ou fixo (com altitude, latitude e longitude).
6. Configuração do servidor MQTT do Brasil ([https://www.meshbrasil.com](https://www.meshbrasil.com/)).
7. Configuração de uplink e downlink do canal primário para enviar mensagens entre o servidor MQTT e a rede mesh.

## Utilização da ferramenta
Para utilizar a ferramenta, visite:
https://tavancini.github.io/meshtastic-br-setup/

## Setup do projeto
1. Instale as dependências:
```sh
npm install
```

2. Para desenvolver localmente, execute:

```sh
npm run dev
```

3. Para gerar os arquivos estáticos (HTML, CSS e Js), execute:

```sh
npm run build
```
