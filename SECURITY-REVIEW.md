# Revisao de seguranca — Railsense

## Escopo

Analise estatica das configuracoes Docker Compose, backend, broker MQTT e ponte
de camera. O firmware e a validacao com ESP32/celular IPCam foram excluidos. Os
achados abaixo descrevem riscos em uma demonstracao acessivel pela rede; o
impacto e menor quando os servicos ficam restritos ao computador local.

## Resumo dos achados

| # | Severity | File | Lines | Vulnerability | Confidence |
|---|----------|------|-------|---------------|------------|
| 1 | 🟡 MEDIUM | `broker/mosquitto.conf`, `docker-compose.yml` | `mosquitto.conf:1-3`; `docker-compose.yml:47-48` | Broker MQTT aceita conexoes anonimas e a porta 1883 e publicada em todas as interfaces; clientes alcancaveis podem injetar leituras e comandos de captura. | 9/10 |
| 2 | 🟡 MEDIUM | `docker-compose.yml`, `backend/src/main/resources/application.yml` | `docker-compose.yml:37,40-48`; `application.yml:57-62` | Cadastro publico fica habilitado por padrao no Compose; qualquer cliente que alcance o backend pode criar uma conta ENGENHEIRO, autorizada a ler a API de monitoramento. | 9/10 |
| 3 | 🟡 MEDIUM | `backend/src/main/java/br/com/railsense_backend/auth/AuthController.java` | `AuthController.java:131` | O rate limiting confia diretamente em `X-Forwarded-For`; um cliente pode alterar o cabecalho e contornar o limite de tentativas em login e cadastro. | 8/10 |
| 4 | 🟢 LOW | Configuracao e testes do backend | `application.yml`; arquivos Java e testes listados abaixo | Existem warnings de null-safety no compilador e o teste de contexto depende de PostgreSQL local, o que pode mascarar falhas na execucao da suite. | 9/10 |

## Detalhes e recomendacoes

### 1. Broker MQTT anonimo e exposto

`broker/mosquitto.conf` permite conexoes anonimas, e o Compose publica a porta
1883 sem restringir o endereco de escuta do host. O backend consome mensagens
de vibracao em `backend/src/main/java/br/com/railsense_backend/mqtt/VibracaoMqttListener.java`;
a ponte de camera subscreve ao topico de captura em
`camera/camera_bridge.py`. Assim, um cliente com acesso ao broker pode enviar
leituras falsas ou disparar capturas.

**Acao para o TCC agora:** sem os dispositivos conectados, nao publique a porta
1883 no host. Se o broker for necessario para o backend no Compose, ele pode
continuar acessivel internamente pela rede Docker. Se a demonstracao nao usar
MQTT, desative a integracao nesse ambiente.

**Antes de conectar dispositivos ou compartilhar o broker:** configurar
autenticacao e ACLs por topico; usar TLS se o trafego sair do host/rede local.

### 2. Cadastro publico habilitado no Compose

O Compose usa `REGISTRATION_ENABLED=true` quando a variavel nao e informada.
O endpoint de cadastro e publico (`AuthController.java:84-115`) e cria contas
com papel `ENGENHEIRO`; `SecurityConfig.java:58-70` permite que esse papel leia
rotas `/api/**`. A configuracao de Render ja desativa o cadastro publico.

**Acao para o TCC agora:** em demonstracao numa rede compartilhada, definir
`REGISTRATION_ENABLED=false` e criar contas de teste controladas. Manter o
cadastro aberto apenas se for deliberado e a demonstracao estiver restrita ao
computador local.

### 3. Validacao de origem no rate limiting

O login e o cadastro limitam tentativas usando o endereco recebido em
`X-Forwarded-For`, sem validar se o proxy que encaminhou a requisicao e
confiavel. Um cliente pode enviar valores falsos e alterar a chave usada pelo
limite de tentativas.

**Acao recomendada:** aceitar `X-Forwarded-For` apenas por um
`ForwardedHeaderFilter` configurado com proxies conhecidos. Em redes sem proxy,
usar o endereco remoto da conexao.

### 4. Warnings de compilacao e confiabilidade dos testes

A analise do backend encontrou cinco warnings de null-safety:

- `backend/src/main/java/br/com/railsense_backend/config/SecurityConfig.java:110`
  usa `String::trim` em um fluxo potencialmente nao anotado como nao nulo.
- `backend/src/main/java/br/com/railsense_backend/service/ImageService.java:84`
  usa `Alerta::getId`.
- `backend/src/main/java/br/com/railsense_backend/auth/SessaoAtualizadaFilter.java:40`
  usa `Usuario::isAtivo`.
- `backend/src/main/java/br/com/railsense_backend/auth/AuthController.java:121`
  usa `Usuario::isAtivo`.
- `backend/src/test/java/br/com/railsense_backend/service/AlertaServiceTest.java:80`
  e `:104` usam referencias de metodo em assertions.

Substituir os casos afetados por lambdas explicitas e garantir que Optional
repositorios e entidades nunca sejam usados como nulos. A suite Maven executou
82 testes: 81 passaram e apenas o teste `contextLoads` falhou porque PostgreSQL
nao estava disponivel em `localhost:5432`. Configurar PostgreSQL no ambiente de
testes ou usar Testcontainers.

## Pontos adicionais do backend

- Em `backend/src/main/resources/application.yml`, o fallback de
  `REGISTRATION_ENABLED` esta como `true`; alterar o padrao para `false` e
  habilitar o cadastro publicamente apenas por configuracao explicita.
- Remover a senha padrao `12345678` do PostgreSQL, exigir credenciais de
  administrador na inicializacao e manter `SESSION_COOKIE_SECURE=true` em HTTPS.
- Definir uma lista restrita de origens CORS por ambiente.
- Revisar e testar todos os endpoints de alertas, ordens, imagens, autenticacao,
  alteracao de papeis e desativacao de contas.
- Se algum endpoint usar DELETE, incluir o metodo na configuracao CORS.

## Controles adicionais antes de uma demonstracao em rede

- Vincular as portas publicadas do PostgreSQL, backend, broker e frontend a
  `127.0.0.1` quando nao precisarem ser acessadas por outros dispositivos.
- Nao reutilizar o fallback local `admin/12345678` de
  `backend/src/main/resources/application.yml` fora do desenvolvimento local;
  definir credenciais unicas por variaveis de ambiente.
- Manter `SESSION_COOKIE_SECURE=false` somente para testes locais por HTTP; em
  acesso HTTPS, habilitar o atributo `Secure`. O blueprint de Render ja o faz.

## Limites

Esta revisao e estatica. Nao foram testados fluxos com hardware, implantacoes
reais, exposicao efetiva de portas, nem comportamento operacional do broker ou
da camera. As configuracoes de producao podem diferir das configuracoes locais.
