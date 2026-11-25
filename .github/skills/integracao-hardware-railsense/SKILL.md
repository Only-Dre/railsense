---
name: integracao-hardware-railsense
description: Use when connecting, configuring, debugging, or validating the Railsense ESP32 sensor, USB-to-MQTT bridge, or phone IP camera.
---

# Integracao de hardware do Railsense

Guie a integracao do ESP32 e da camera do celular com o backend RailSense. Os
dispositivos podem estar indisponiveis; nesse caso, faca apenas verificacoes
estaticas e de compilacao, e identifique claramente os testes fisicos pendentes.
Nunca declare hardware validado sem observar o teste com o dispositivo.

## Antes de comecar

1. Verifique a branch e o estado do worktree. Preserve alteracoes locais do
   usuario; nao reverta arquivos sem autorizacao.
2. Leia as configuracoes atuais antes de editar:
   - `hardware/platformio.ini` e `hardware/src/main.cpp`
   - `hardware/serial_bridge.py` e `hardware/requirements.txt`
   - `camera/camera_bridge.py` e `camera/.env.example`
   - `mqtt/topics.md`, `broker/mosquitto.conf` e `docker-compose.yml`
   - `backend/src/main/java/br/com/railsense_backend/mqtt/VibracaoMqttListener.java`
3. Confirme com o usuario quais dispositivos e caminho serao testados: ESP32
   via USB, ESP32 via Wi-Fi ou ambos; app e URL da camera; eixo de teste.
4. Nunca solicite que credenciais sejam coladas no chat, logs ou commits. Use
   configuracao local ignorada pelo Git e confirme o estado do rastreamento
   antes de gravar segredos.

## Estado conhecido e pontos a confirmar

- O projeto PlatformIO mais completo fica em `hardware/`. Existe tambem um
  projeto PlatformIO na raiz com outro `src/main.cpp`; nao presuma que sao
  equivalentes nem mova/remova arquivos sem verificar referencias.
- O firmware USB usa o ambiente `esp32dev-usb`; `hardware/serial_bridge.py`
  publica os dados serializados nos mesmos topicos MQTT.
- O firmware Wi-Fi tem configuracoes de rede definidas no proprio codigo. Nao
  presuma que `hardware/include/secrets.h` as fornece: verifique inclusoes e
  referencias no codigo antes de alterar configuracao. O header atualmente
  rastreado pelo Git nao deve receber credenciais reais.
- O broker local permite MQTT anonimo e o Compose publica a porta 1883. Limite
  testes a uma rede privada confiavel; nao abra encaminhamento de portas nem
  desative o firewall. Para uma rede compartilhada, interrompa e solicite
  configuracao de autenticacao e ACLs antes de prosseguir.
- O sensor deve usar o mesmo identificador configurado no cadastro do eixo
  (por exemplo, `esp32-01`). Confirme o valor real no firmware; nao o assuma.
- A ponte da camera envia imagens para `POST /api/imagens`; a conta configurada
  precisa ter papel `ENGENHEIRO`, e `EIXO_ID` deve ser valido.

## Fluxo recomendado: ESP32 por USB

Prefira USB como primeiro teste quando a rede Wi-Fi nao estiver pronta ou nao
permitir que o ESP32 alcance o broker.

1. No diretorio `hardware`, compile sem conectar ou gravar no dispositivo:

   ```powershell
   pio run -e esp32dev-usb
   ```

2. Quando o ESP32 estiver conectado e o usuario confirmar a placa/porta, grave:

   ```powershell
   pio run -e esp32dev-usb -t upload
   ```

   Nao inicie a gravacao se PlatformIO nao detectar uma placa compativel ou se
   a porta nao estiver confirmada.

3. Configure `hardware/.env` localmente com `SERIAL_PORT` (opcional),
   `SERIAL_BAUD` (padrao 115200), `MQTT_BROKER` e `MQTT_PORT`. O broker tem de
   ser o mesmo usado pelo backend.
4. Instale as dependencias num ambiente virtual local e execute a ponte:

   ```powershell
   py -m venv .venv
   .\.venv\Scripts\python -m pip install -r requirements.txt
   .\.venv\Scripts\python serial_bridge.py
   ```

   Execute os comandos a partir de `hardware`. Feche o monitor serial do
   PlatformIO antes: uma porta serial nao pode ser usada simultaneamente por
   dois processos.

5. Verifique a publicacao em `railsense/sensor/vibracao` e a chegada de leituras
   ao backend/eixo. Nao trate apenas a mensagem serial como prova de que MQTT,
   backend e frontend estao integrados.

## Fluxo alternativo: ESP32 via Wi-Fi

1. Confirme que o computador/broker e o ESP32 estao na mesma rede alcançavel e
   que o firewall permite somente o trafego necessario.
2. Verifique a configuracao de SSID, senha e endereco MQTT efetivamente lida
   pelo firmware. Use valores apenas em arquivos locais nao rastreados; se o
   firmware ainda tiver valores embutidos, proponha uma correcao segura antes
   de adicionar credenciais.
3. Compile e grave o ambiente `esp32dev` somente com a placa conectada e
   autorizacao do usuario.
4. Confirme que o ESP32 e o backend usam o mesmo broker e que as mensagens
   aparecem no topico `railsense/sensor/vibracao`.

Nao tente o modo USB e Wi-Fi como se fossem o mesmo caminho: registre qual foi
testado e qual continua pendente.

## Fluxo da camera do celular

1. Confirme o modelo do celular, o app de camera IP, a URL da imagem ou stream,
   e que celular e computador estao na mesma rede local. Nao exponha a camera
   diretamente a internet.
2. Copie `camera/.env.example` para `camera/.env` e preencha localmente
   `CAMERA_URL`, credenciais se necessarias, `BACKEND_URL`, conta backend,
   `EIXO_ID` e intervalo. Nao mostre os valores no chat ou nos logs.
3. Verifique a URL da camera a partir do computador que executara a ponte; a
   URL pode servir JPEG ou stream MJPEG.
4. Instale `camera/requirements.txt` num ambiente virtual local e execute
   `python camera_bridge.py` a partir do diretorio `camera`.
5. Confirme login no backend e upload de uma imagem de teste. Se o teste for
   apenas sob demanda, configure `INTERVALO_SEGUNDOS=0`; com MQTT ativo, valide
   `railsense/camera/capturar` e o evento `railsense/camera/imagem`.
6. Pare a ponte e remova apenas arquivos locais de teste se o usuario pedir.

## Topicos e verificacoes ponta a ponta

Consulte `mqtt/topics.md` como referencia para payloads e fluxos. Os topicos
atuais incluem:

- `railsense/sensor/vibracao`: leitura do sensor, com identificador do
  dispositivo e valor.
- `railsense/camera/capturar`: pedido de captura.
- `railsense/camera/imagem`: confirmacao publicada pela ponte apos upload aceito.

Para considerar uma demonstracao integrada, verifique separadamente:

1. O dispositivo transmite ou a ponte USB le a serial.
2. O broker recebe no topico esperado.
3. O backend aceita e registra a leitura, vinculada ao eixo correto.
4. O pedido de captura alcança a ponte, quando esse caminho estiver ativo.
5. A imagem da camera chega ao backend e aparece no fluxo esperado da aplicacao.

Registre resultados observados, comandos executados e falhas. Distinga
explicitamente compilacao/simulacao de validacao fisica. Se hardware estiver
desconectado, marque os passos correspondentes como **nao testados** e nao
altere configuracoes para simular sucesso.
