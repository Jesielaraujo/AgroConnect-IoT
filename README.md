# AgroConnect-IoT
O AgroConnect IoT é um sistema de automação e monitoramento em tempo real voltado para o segmento de Agricultura de Precisão (Agro), projetado para otimizar o microclima e os recursos hídricos em estufas de cultivo monitoradas. Desenvolvido sobre a plataforma ESP32, o projeto une o conceito de conectividade híbrida (Wi-Fi e Bluetooth) à conteinerização de serviços em nuvem/local utilizando Docker.

## 🛠️ Arquitetura e Funcionamento Tecnológico
O coração do sistema reside no microcontrolador ESP32, que opera de forma multitarefa gerenciando sensores, atuadores e duas redes de comunicação simultâneas:

#### Camada de Coleta e Automação Local (Hardware):

Entradas Analógicas: Monitoramento constante dos índices de luminosidade (via sensor LDR) e temperatura.

Entrada Digital: Um botão físico configurado com lógica toggle atua como o interruptor geral de segurança do sistema.

Saídas Digitais: Um Relé de potência aciona os sistemas pesados de climatização/irrigação (exaustores/bombas d'água) e um Buzzer ativo emite alertas sonoros para indicar a inicialização do circuito.

Saída Analógica (PWM): Um LED atuador varia sua intensidade luminosa via modulação por largura de pulso (analogWrite), brilhando a 0% (sistema desligado), 10% (estado de espera) e 100% (estado ativo com o relé ligado).

### Conectividade e Protocolos de Comunicação:

Comunicação Bluetooth (Classic): Cria um canal direto ponto a ponto com dispositivos móveis em campo. Através de um protocolo de comandos específico (ligar, desligar, temp=XX, luz=XX), o operador consegue redefinir os limites de acionamento do relé dinamicamente sem alterar o código-fonte.

Comunicação Wi-Fi & NTP: O ESP32 conecta-se à rede local e sincroniza o horário exato através de servidores NTP para realizar o registro temporal preciso dos eventos.

### Fluxo de Dados e Infraestrutura em Containers (Docker):

A cada 2 segundos, o ESP32 empacota as leituras físicas e as variáveis do Bluetooth em uma string JSON estruturada, publicando-a via protocolo MQTT para um Broker (Eclipse Mosquitto).

O Node-RED consome esse tópico MQTT, processa a string e faz a persistência dos dados em um Banco de Dados Relacional (PostgreSQL/MySQL).

Toda a infraestrutura do servidor (Broker, Node-RED e Banco de Dados) é instanciada em microsserviços isolados através do Docker Desktop.

Por fim, o Grafana consome o banco de dados para renderizar um Dashboard em tempo real, plotando gráficos que refletem tanto o histórico dos sensores quanto as alterações de limites enviadas remotamente pelo Bluetooth.

## 🎯 Resumo dos Critérios Atendidos
Segmento: Domótica / Agro (Monitoramento de Estufas).

Variáveis Digitais: 3 (1 Entrada: Botão | 2 Saídas: Relé e Buzzer).

Variáveis Analógicas: 3 (2 Entradas: Temperatura e LDR | 1 Saída: LED PWM).

Infraestrutura: Docker Desktop, Node-RED, MQTT, Banco Relacional e Grafana.

Controle: Protocolo de comandos Bluetooth customizado.
