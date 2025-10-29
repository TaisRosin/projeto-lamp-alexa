### Integrantes da equipe

- Mariana de Souza
- Matheus Henrique Albert
- Taiane Queiros Rufatto
- Taís Alessandra Rosin

---

### Objetivo

Integrar um dispositivo (uma lâpada) a ALEXA por meio de uma plataforma SINRIC. Usaremos Wi-Fi e comando de voz.

---

### Materiais necessários

- 2 metros de fio de 1,5 mm² (fio padrão tomada)
- Plug macho tomada (10 A)
- Lâmpada
- Soquete bocal de lâmpada
- Um plug de tomada fêmea de 10 A (caso queira ligar um eletrodoméstico, como ventilador, climatizador, motor, etc)
- Um modulo relé / relay
- Microcontrolador NodeMCU ESP8266

---

### Plataformas e extensões utilizadas

- *VScode*
    - *extensão:* PlatformIO
- *Sinric Pro* (para fazer a integração com a alexa)

---

### Primeiros passos

### 1 - Crie uma conta

https://portal.sinric.pro/login

### 2 - Adicione um dispositivo

![img-adicionar_dispositiv.png](https://i.imgur.com/abcd1234.png)

### 3- Preencha informações do dispositivo

### 4- Ative as seguintes notificações

### 5- Salvar as configurações

### 6- Veja os dispositivos ativos

---

### PlatformIO — Códigos

após seguir os primeiros passos, adicionando e configurando o dispositivo, adicione o seguinte código no arquivo *platformio.ini*

**— Integração com a alexa**

jsx
lib_deps = https://github.com/sinricpro/esp8266-esp32-sdk

// Inclui as bibliotecas necessárias
#include <Arduino.h>
#include <ESP8266WiFi.h>
#include <SinricPro.h>
#include <SinricProSwitch.h>

// Define constantes para conexão Wi-Fi e SinricPro
#define WIFI_SSID "xxxxxxxxx"
#define WIFI_PASS "xxxxxxxxxxxx"
#define APP_KEY "xxxxxxxxxx"
#define APP_SECRET "xxxxxxxxxxxx"
#define SWITCH_ID "xxxxxxxxxx"
#define BAUD_RATE 9600

// Define os pinos GPIO para o botão e o rele
#define BUTTON_PIN 0 // GPIO para o BOTÃO (inverso: LOW = pressionado, HIGH =
liberado)
#define RELE_PIN 5
bool myPowerState = false; // Indica o estado atual do dispositivo (ligado/desligado)
unsigned long lastBtnPress = 0; // Última vez que o botão foi pressionado (usado para
debounce)

// Esta função é chamada quando o dispositivo é controlado remotamente via SinricPro
bool onPowerState(const String &deviceId, bool &state) {
// Exibe no Serial Monitor o estado atual
Serial.printf("Device %s turned %s (via SinricPro) \r\n", deviceId.c_str(), state?"on":"off");

myPowerState = state;
// Define o estado do rele com base no estado do dispositivo (Operador Ternário)
digitalWrite(RELE_PIN, myPowerState?LOW:HIGH);
return true;
}

// Esta função trata a pressão do botão físico
void handleButtonPress() {
unsigned long actualMillis = millis();
if (digitalRead(BUTTON_PIN) == LOW && actualMillis - lastBtnPress > 1000) {
myPowerState = !myPowerState;
digitalWrite(RELE_PIN, myPowerState?LOW:HIGH);
// Envia o novo estado para o servidor SinricPro
SinricProSwitch& mySwitch = SinricPro[SWITCH_ID];
mySwitch.sendPowerStateEvent(myPowerState);
Serial.printf("Device %s turned %s (manually via flashbutton)\r\n",
mySwitch.getDeviceId().c_str(), myPowerState?"on":"off");
lastBtnPress = actualMillis;
}
}

// Função para conectar o ESP8266 ao Wi-Fi
void setupWiFi() {
Serial.printf("\r\n[Wifi]: Connecting");
WiFi.begin(WIFI_SSID, WIFI_PASS);
while (WiFi.status() != WL_CONNECTED) {
Serial.printf(".");
delay(250);
}
Serial.printf("connected!\r\n[WiFi]: IP-Address is %s\r\n", WiFi.localIP().toString().c_str());
}

// Função para configurar a conexão com SinricPro
void setupSinricPro() {
SinricProSwitch& mySwitch = SinricPro[SWITCH_ID];
mySwitch.onPowerState(onPowerState);
// Configura callbacks para a conexão com SinricPro
SinricPro.onConnected({ Serial.printf("Connected to SinricPro\r\n"); });
SinricPro.onDisconnected({ Serial.printf("Disconnected from SinricPro\r\n"); });
SinricPro.begin(APP_KEY, APP_SECRET);
}

// Função principal de configuração
void setup() {
pinMode(BUTTON_PIN, INPUT_PULLUP);
pinMode(RELE_PIN, OUTPUT);
digitalWrite(RELE_PIN, HIGH);
Serial.begin(BAUD_RATE);

setupWiFi();
setupSinricPro();
}

// Loop principal
void loop() {
handleButtonPress();
SinricPro.handle(); // Constantemente verifica e trata solicitações da plataforma SinricPro
}



- depois teste a aplicação no próprio painel do Sinric PRO.

---

### Aplicativo Amazon Alexa

baixe o aplicativo Amazon Alexa no seu celular

- agora crie suas rotinas e seus comandos :)

---

### Como o site controla a lâmpada

- quando o microcontrolador é ligado, ele se conecta à rede wi-fi e cria um servidor web
- o servidor é tipo um “Mini site” que a gente acessa digitando o endereço IP dele no navegador
- no site, vai aparecer dois botões. Um de ligar e outro para desligar
    - quando clica em um deles:
    1. O site manda um comando pro ESP8266 (por exemplo, “/on” para ligar).
    2. O ESP recebe esse comando e envia um sinal elétrico pro *relé*.
    3. O relé funciona como um *interruptor*, que liga ou desliga a energia da lâmpada.

---

### Como funciona o controle por voz (Alexa)

Além do site, o projeto também permite ligar a lâmpada falando com a Alexa.

Pra isso, usamos o *Sinric Pro*, que é uma plataforma que faz a “ponte” entre a Alexa e o nosso ESP8266.

O processo é assim:

1. Criamos uma conta no site do Sinric Pro e registramos um *dispositivo virtual* chamado “Lâmpada”.
2. No código do ESP8266, colocamos as *chaves de acesso* do Sinric Pro (essas chaves ligam o nosso dispositivo real ao virtual).
3. O ESP se conecta à internet e fica “ouvindo” o que o Sinric Pro manda.
4. Quando falamos “Alexa, ligar a lâmpada”, a Alexa manda esse comando pro Sinric Pro.
5. O Sinric Pro repassa o comando pro ESP8266.
6. O ESP aciona o relé e *liga a lâmpada de verdade*.

---

### Protocolo de comunicação utilizado é o WebSocket

- O *Sinric Pro* se comunica com o *ESP8266* por meio do *protocolo WebSocket, que é uma tecnologia baseada na web usada para manter uma **conexão contínua e bidirecional* entre o dispositivo e o servidor.
- Diferente do protocolo *HTTP tradicional* (usado no site local), onde o cliente precisa fazer uma nova requisição toda vez que quer uma resposta, o *WebSocket* cria uma *conexão permanente*
- Assim, o servidor (Sinric Pro) pode *enviar mensagens instantâneas* ao ESP8266 *sem precisar que ele fique pedindo informações o tempo todo*

---

### Como o ele se conecta com a alexa

- O *ESP8266* se conecta à *Alexa* por meio da *plataforma Sinric Pro*, que funciona como uma ponte entre eles.
- Primeiro, o ESP se conecta ao *Wi-Fi* e depois ao *servidor do Sinric Pro* usando o *protocolo WebSocket*.
- Quando o usuário diz “*Alexa, ligar a lâmpada*”, a Alexa envia o comando para o Sinric Pro, que repassa a mensagem ao ESP8266.
- O ESP recebe o comando e aciona o *relé*, ligando ou desligando a lâmpada em tempo real.

---

### Formato de comunicação

- A comunicação é *bidirecional, porque o **ESP8266* e o *Sinric Pro* trocam informações nos dois sentidos:
- a Alexa envia comandos para o ESP (*ligar/desligar) e o ESP também envia de volta seu estado atual (ligado/desligado*) para atualizar o sistema em tempo real.
