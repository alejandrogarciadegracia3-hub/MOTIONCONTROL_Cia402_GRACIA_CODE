CONTROLADOR CNC RELEASE 9


Este trabalho é apresentado aos profissionais que concebem, desenvolvem e evoluem as tecnologias empregadas na automação industrial: arquitetos de software para CLPs, desenvolvedores de sistemas de Motion Control, projetistas de servoacionamentos, especialistas em tempo real, pesquisadores e engenheiros envolvidos na criação de plataformas de controle industrial.

A proposta consiste na apresentação de uma arquitetura de software desenvolvida integralmente em Texto Estruturado (IEC 61131-3), cuja organização separa as etapas de interpretação, processamento e preparação dos dados da execução determinística do controle de movimento. Nessa arquitetura, programas textuais são interpretados previamente, convertidos em estruturas de dados otimizadas e armazenados em buffers, permitindo que a camada responsável pela comunicação com os servoacionadores execute apenas operações compatíveis com os rigorosos requisitos temporais do sistema.

O trabalho integra conceitos de interpretação de linguagem, controle numérico computadorizado, gerenciamento de buffers, comunicação industrial, manipulação dos objetos do perfil CiA 402 e sincronização em redes EtherCAT, reunindo esses elementos em uma arquitetura única voltada à execução em controladores programáveis industriais.

Mais do que apresentar uma implementação, este trabalho propõe uma arquitetura passível de análise técnica. O convite é dirigido aos profissionais que constroem as tecnologias utilizadas pela indústria para que avaliem seus fundamentos, discutam suas decisões de projeto e examinem suas possibilidades de aplicação, evolução e integração com plataformas modernas de automação industrial.

Toda análise técnica, crítica fundamentada ou contribuição proveniente da experiência de profissionais envolvidos no desenvolvimento de sistemas industriais será considerada uma oportunidade para o aprimoramento desta arquitetura e para a ampliação do conhecimento na área de controle de movimento e automação industrial.



MOTIONCONTROL_Cia402_GRACIA_CODE

├── Controlador CNC em ST
│   ├── Interpretador GRACIA_CODE
│   ├── Motion Control
│   └── CiA 402 / EtherCAT

├── HMI Web
│   ├── HTML
│   ├── Node.js
│   └── WebSocket

├── FENIX
│   ├── Python
│   └── Tkinter

└── Documentação
┌──────────────────────────────────────────────────────────────────┐
│  [1] ENTRADA / HMI                              (fora do tempo real)
│  ─────────────────────────────────────────────────────────────── │
│  HMI Python/Tkinter (Android)                                     │
│  USB serial (/dev/ttyGS0) · protocolo texto "nome=valor"          │
│  → HMI_USB_Receiver_Transmitter → ProcessarLinha                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  [2] CARREGAMENTO DO PROGRAMA                                     │
│  ─────────────────────────────────────────────────────────────── │
│  Loader_ProgGCODE → ProgramaGCODE[1..200]                          │
│  rxDone → dispara Interpretador                                   │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  [3] INTERPRETAÇÃO                          (GRACIA_code → params) │
│  ─────────────────────────────────────────────────────────────── │
│  Interpretador_GRACIA_CODE                                         │
│  Blocos "Função: Fx" ... "$"                                       │
│  Dicionário: Variaveis[] / TiposVariaveis[]                        │
│  StrToReal · StrToInt · StrSplit                                   │
│  ExecuteFunc → roteia F1–F6                                        │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
╔════════════════════════════════════════════════════════════════╗
║  [4] GERAÇÃO GEOMÉTRICA                                          ║
║      task LENTA · não determinística · até ~15s (sem problema)   ║
║  ──────────────────────────────────────────────────────────────  ║
║   F1 Linear/Circular      F3 Bi-Planar                            ║
║   F4 Bi-Planar Pumpkin    F5 Cônica       F6 Revolution Points     ║
║                                                                    ║
║   cada uma calcula:                                                ║
║     • geometria (trigonometria, discretização angular)             ║
║     • CalcAccelDecelFromProfile → aceleração máxima real           ║
║       (a partir de Torque / Massa / Inércia)                       ║
║     • VELOCIDADE_X_Y_Z                                             ║
║     • ANG1 → zona de frenagem antecipada (look-ahead)              ║
║     • ConvertRealDiffToTargetDINT                                  ║
║                                                                    ║
║   grava em → bufferX/Y/Z · bufferVelX/Y/Z · bufferTarget_H/L      ║
║   (buffer único, global, sequencial — sem reentrância)             ║
║   JOG usa buffer exclusivo à parte                                 ║
╚════════════════════════════════════════════════════════════════╝
                              │
              ═══════ SINCRONIZAÇÃO ENTRE TASKS ═══════
                              │
                              ▼
╔════════════════════════════════════════════════════════════════╗
║  [5] EXECUÇÃO DETERMINÍSTICA                                     ║
║      task RÁPIDA · ciclo EtherCAT · 1 ms                         ║
║  ──────────────────────────────────────────────────────────────  ║
║   DRIVE_INPUT_OUTPUT                                              ║
║     lê buffer ponto a ponto (já calculado)                        ║
║     handshake CiA 402:                                            ║
║       escreve alvo (607Ah)                                        ║
║       → seta NEW_SETPOINT (6040h)                                 ║
║       → aguarda ACKNOWLEDGE (6041h)                                ║
║       → limpa bit → avança                                        ║
║     sincronismo X/Y/Z via readyToAdvanceGlobal                     ║
║                                                                    ║
║   JOG_X_Y_Z_PULSADO / ControlPushButtons                          ║
║     comando manual direto, em paralelo, bypassa o buffer           ║
╚════════════════════════════════════════════════════════════════╝
                              │
                              ▼   (mesmo ciclo rápido, em paralelo)
╔════════════════════════════════════════════════════════════════╗
║  [6] MALHA DE COMPENSAÇÃO                                         ║
║      camada de OBSERVAÇÃO / PESQUISA — não crítica                ║
║      (o drive já é confiável de forma autônoma)                   ║
║  ──────────────────────────────────────────────────────────────  ║
║   LAG_6064_TORQUE_FORCA_VELOCIDADE                                 ║
║     posição real (6064h) vs. posição prevista no buffer            ║
║     → calcula erro de seguimento (lag) por eixo                    ║
║     → ajusta 6081h se erro exceder limite                          ║
║     → aciona Quick Stop se correção insuficiente                   ║
╚════════════════════════════════════════════════════════════════╝
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  [7] RETROALIMENTAÇÃO HMI                                          │
│  ─────────────────────────────────────────────────────────────── │
│  PrepararLinhaTransmissao → posição, lag, estados de falha → HMI  │
└──────────────────────────────────────────────────────────────────┘



![Imagem 1](20260708_183033.jpg)

![Imagem 2](20260708_225320.jpg)

![Imagem 3](20260708_230813.jpg)

![Imagem 4](20260708_183108.jpg)

![Imagem 5](20260708_183140.jpg)

![Imagem 6](20260708_183203.jpg)

![Imagem 7](20260708_183230.jpg)

![Imagem 8](20260708_183301.jpg)
