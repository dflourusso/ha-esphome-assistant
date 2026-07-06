# Porteiro Garagem — Home Assistant

Automação de recepção na garagem com **DFLTech Assistant** (voz) e agente OpenAI **Porteiro**.

**Instalação:** copiar e colar cada YAML na interface do HA — **sem** editar `configuration.yaml`.

## Pré-requisitos

1. ESPHome **dfltech-assistant** na garagem (satellite de voz).
2. Agente de conversa **Porteiro** com **Assist desligado** — cole o texto de [`prompt-porteiro.txt`](prompt-porteiro.txt).
3. Lista **Entregas** (`todo.entregas`): `summary` = código; na descrição use `descrição:` e `palavra-chave:`.
4. Pipeline no satellite: STT + TTS (para `assist_satellite.ask_question`).
5. **AI Task** configurado (Settings → AI) — usado pelo script 07 para leitura pela câmera.

## Entidades usadas

| Entidade | Uso |
|----------|-----|
| `assist_satellite.garagem_dfltech_assistant_assist_satellite` | Perguntas e respostas faladas |
| `button.garagem_dfltech_assistant_start_conversation` | Parar conversa |
| `conversation.porteiro` | Agente OpenAI (Assist desligado) |
| `todo.entregas` | Códigos de entrega |
| `camera.campainha_generic` | Câmera da portaria (script 07) |
| `switch.portao_garagem` | **Placeholder** — troque no script 05 |
| `notify.mobile_app_SEU_TELEFONE` | **Placeholder** — troque no script 04 |

## Instalação (colar na UI)

Ordem importa: scripts primeiro, automação por último.

### Scripts

Em **Configurações → Scripts → Criar script → ⋮ → Editar em YAML**, cole o conteúdo de cada arquivo.

**Importante:** na UI **não use** a chave `id:` (só vale em `configuration.yaml`). O `entity_id` é gerado a partir do `alias` — use exatamente os aliases dos arquivos abaixo.

| Ordem | Arquivo | Alias (gera o entity_id) |
|-------|---------|--------------------------|
| 1 | [`scripts/01-porteiro-aguardar-idle.yaml`](scripts/01-porteiro-aguardar-idle.yaml) | `porteiro aguardar idle` → `script.porteiro_aguardar_idle` |
| 2 | [`scripts/02-porteiro-parar-conversa.yaml`](scripts/02-porteiro-parar-conversa.yaml) | `porteiro parar conversa` → `script.porteiro_parar_conversa` |
| 3 | [`scripts/03-porteiro-buscar-codigo.yaml`](scripts/03-porteiro-buscar-codigo.yaml) | `porteiro buscar codigo` → `script.porteiro_buscar_codigo` |
| 4 | [`scripts/04-porteiro-notificar-morador.yaml`](scripts/04-porteiro-notificar-morador.yaml) | `porteiro notificar morador` → `script.porteiro_notificar_morador` |
| 5 | [`scripts/05-porteiro-abrir-portao.yaml`](scripts/05-porteiro-abrir-portao.yaml) | `porteiro abrir portao` → `script.porteiro_abrir_portao` |
| 6 | [`scripts/06-porteiro-atender.yaml`](scripts/06-porteiro-atender.yaml) | `porteiro atender` → `script.porteiro_atender` |
| 7 | [`scripts/07-porteiro-ler-codigo-camera.yaml`](scripts/07-porteiro-ler-codigo-camera.yaml) | `porteiro ler codigo camera` → `script.porteiro_ler_codigo_camera` |
| 8 | [`scripts/08-porteiro-confirmar-entrega.yaml`](scripts/08-porteiro-confirmar-entrega.yaml) | `porteiro confirmar entrega` → `script.porteiro_confirmar_entrega` |

Salve cada script antes de criar o próximo. Depois de salvar, confira em **Ferramentas de desenvolvedor → Estados** se o `entity_id` bate com a tabela.

**Entity ID:** o nome na lista pode ser `03 porteiro_buscar_codigo`, mas o **ID da entidade** (⚙️ no script) tem que ser exatamente `porteiro_buscar_codigo` — sem `03_` no início. O script 06 chama `script.porteiro_buscar_codigo`; se o ID for `script.03_porteiro_buscar_codigo`, dá *not found*.

**Recarregar:** após corrigir scripts, vá em **Ferramentas de desenvolvedor → YAML** → **Scripts** → *Recarregar*. Confira se os `script.porteiro_*` aparecem em Estados e não estão `unavailable`.

**Por que `porteiro parar conversa` no início?** O firmware do ESP pode iniciar o Assist na campainha. Esse passo cancela uma conversa em andamento antes do porteiro falar. Se o satellite já estiver `idle`, não faz nada e não espera.

**Por que `porteiro aguardar idle` no loop?** Entre uma pergunta e outra o satellite passa por listening/processing. Só espera se ainda não estiver `idle` (evita os 120s quando já está livre).

### Automação (trigger livre)

O porteiro **não depende da campainha**. Qualquer botão, interruptor ou cena pode chamar `script.porteiro_atender`.

Em **Configurações → Automações → Criar automação → ⋮ → Editar em YAML**, cole [`automation-porteiro-iniciar.yaml`](automation-porteiro-iniciar.yaml) e troque o trigger pelo seu dispositivo.

Exemplos de trigger:

| Dispositivo | Trigger |
|-------------|---------|
| `input_button` | `state` → `to: pressed` |
| `switch` / interruptor | `state` → `to: "on"` |
| Cena / botão de automação | `event` `call_service` com `scene.turn_on` |
| Campainha ESPHome | `event` `esphome.dfltech_assistant_doorbell` (opcional) |
| Manual / dashboard | botão que executa `script.porteiro_atender` |

Ao iniciar, o script fala **Bom dia / Boa tarde / Boa noite** (conforme o horário) + **Em que posso ajudar?** e segue o loop de conversa.

## Arquitetura: IA vs script

| Responsabilidade | Quem |
|------------------|------|
| Conversa natural, classificar intenção, extrair código da fala | Agente **Porteiro** (`prompt-porteiro.txt`) |
| Contar tentativas falhas de código (via histórico `conversation_id`) | Agente **Porteiro** |
| Decidir voz vs câmera, quando avisar o morador após 3 falhas | Agente **Porteiro** |
| Validar código na lista, ler câmera, abrir portão, notificar push | Scripts HA |

O agente responde sempre em JSON (`fala`, `tipo`, `codigo`, `metodo_codigo`, `resumo`, `encerrar`). O script 06 interpreta o JSON e executa ações — mensagens `[Sistema: ...]` informam falhas de validação sem lógica de contagem no YAML.

## Tipos de atendimento

| Tipo | Comportamento |
|------|----------------|
| **visita** | Conversa natural; coleta nome, quem visita e motivo; `resumo` + `encerrar: true` → notifica morador |
| **geral** | Prestador de serviço, técnico, etc.; avisa o morador como visita (não dispensa) |
| **entrega — encomenda** | Pede código de entrega/rastreio (voz ou câmera); valida em `todo.entregas` |
| **entrega — comida** | iFood, Rappi, etc.; **sem código**; `resumo` + `encerrar: true` → notifica morador |
| **vendedor** | Propaganda/ambulante; dispensa educadamente **sem** avisar morador |

## Fluxo de entrega (encomenda com código)

### Código validado

1. Agente extrai o código da fala (`metodo_codigo: voz`, campo `codigo`) **ou** aciona a câmera (`metodo_codigo: camera`).
2. **Voz:** `codigo` no JSON → `script.porteiro_confirmar_entrega`.
3. **Câmera:** instruções ao entregador → espera → `script.porteiro_ler_codigo_camera` (AI Task) → confirma entrega com o código lido.
4. `script.porteiro_buscar_codigo` compara com o **summary** de cada item em `todo.entregas`.
5. Se achar: marca concluído, notifica morador, **avisa o entregador por voz** (com palavra-chave se houver) e abre o portão.
6. Encerra o atendimento (`stop`) — não volta a pedir código no loop.

### Código não encontrado (antes da 3ª falha)

1. Script envia `[Sistema: o código X não foi encontrado]` (ou falha na câmera) ao agente, mantendo o `conversation_id`.
2. Agente conta a falha no histórico e pede nova tentativa — por voz ou mostrando o pacote à câmera.
3. Loop continua até validar ou chegar à 3ª falha.

### Após 3 falhas de validação

1. Agente decide avisar o morador: `resumo` + `encerrar: true`, `codigo: null`.
2. Script notifica o morador, fala com o entregador para aguardar e encerra.

## Ajustes manuais

- **Portão:** edite `switch.portao_garagem` em `05-porteiro-abrir-portao.yaml` (ou no script já salvo no HA).
- **Câmera:** `camera.campainha_generic` em `07-porteiro-ler-codigo-camera.yaml`.
- **AI Task:** configure em Settings → AI (sub-entry AI Task no OpenAI/Google/etc.). O script 07 usa `ai_task.generate_data` sem `entity_id` fixo (usa o preferido do HA).
- **Notificação:** edite `notify.mobile_app_SEU_TELEFONE` no script 04.
- **Agente:** deve existir como `conversation.porteiro` (cole o prompt de `prompt-porteiro.txt`, Assist desligado). Atualize o prompt sempre que alterar `prompt-porteiro.txt` neste repositório.
- **Satellite:** ajuste `assist_satellite.garagem_dfltech_assistant_assist_satellite` em `06-porteiro-atender.yaml` se o entity_id do seu ESP for outro.
- **Horário da saudação:** 05h–11h59 bom dia, 12h–17h59 boa tarde, resto boa noite (edite em `06-porteiro-atender.yaml`).
