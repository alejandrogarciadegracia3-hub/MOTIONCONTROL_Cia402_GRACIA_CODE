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
flowchart TD
    A["1. ENTRADA / HMI<br/>fora do tempo real<br/>HMI Python/Tkinter Android · USB serial<br/>HMI_USB_Receiver_Transmitter → ProcessarLinha"]
    B["2. CARREGAMENTO DO PROGRAMA<br/>Loader_ProgGCODE → ProgramaGCODE[1..200]<br/>rxDone → dispara Interpretador"]
    C["3. INTERPRETAÇÃO<br/>GRACIA_code → parâmetros<br/>Interpretador_GRACIA_CODE · ExecuteFunc<br/>StrToReal · StrToInt · StrSplit"]

    subgraph GERACAO["4. GERAÇÃO GEOMÉTRICA — task LENTA · não determinística · até ~15s (ok)"]
        D1["F1 Linear/Circular"]
        D2["F3 Bi-Planar"]
        D3["F4 Bi-Planar Pumpkin"]
        D4["F5 Cônica"]
        D5["F6 Revolution Points"]
        D6["CalcAccelDecelFromProfile<br/>Torque · Massa · Inércia"]
        D7["ANG1 — zona de frenagem<br/>antecipada (look-ahead)"]
        D8["Buffers: bufferX/Y/Z ·<br/>bufferVelX/Y/Z · bufferTarget_H/L<br/>(único, global, sequencial)"]
        D1 --> D6 --> D7 --> D8
        D2 --> D6
        D3 --> D6
        D4 --> D6
        D5 --> D6
    end

    SYNC{{"SINCRONIZAÇÃO ENTRE TASKS"}}

    subgraph EXEC["5. EXECUÇÃO DETERMINÍSTICA — task RÁPIDA · EtherCAT · 1 ms"]
        E1["DRIVE_INPUT_OUTPUT<br/>lê buffer ponto a ponto"]
        E2["Handshake CiA 402<br/>607Ah → NEW_SETPOINT(6040h)<br/>→ ACKNOWLEDGE(6041h) → avança"]
        E3["Sincronismo X/Y/Z<br/>readyToAdvanceGlobal"]
        E4["JOG_X_Y_Z_PULSADO /<br/>ControlPushButtons<br/>(comando manual, bypassa buffer)"]
        E1 --> E2 --> E3
    end

    subgraph LAG["6. MALHA DE COMPENSAÇÃO — observação/pesquisa, não crítica"]
        F1L["LAG_6064_TORQUE_FORCA_VELOCIDADE<br/>posição real (6064h) vs. prevista"]
        F2L["ajusta 6081h se erro > limite"]
        F3L["Quick Stop se correção insuficiente"]
        F1L --> F2L --> F3L
    end

    G["7. RETROALIMENTAÇÃO HMI<br/>PrepararLinhaTransmissao<br/>posição, lag, falhas → HMI"]

    A --> B --> C --> GERACAO --> SYNC --> EXEC
    EXEC -.paralelo.-> LAG
    EXEC --> G
    LAG --> G

![Imagem 1](20260708_183033.jpg)

![Imagem 2](20260708_225320.jpg)

![Imagem 3](20260708_230813.jpg)

![Imagem 4](20260708_183108.jpg)

![Imagem 5](20260708_183140.jpg)

![Imagem 6](20260708_183203.jpg)

![Imagem 7](20260708_183230.jpg)

![Imagem 8](20260708_183301.jpg)
