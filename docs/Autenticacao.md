# Tipos de Autenticação

### Autenticação EAP-TLS

```mermaid
sequenceDiagram
    participant S as Suplicante
    participant A as Autenticador
    participant R as Servidor RADIUS
    participant D as Diretório / CA

    Note over S,A: Porta inicia no estado não autorizado
    S->>A: EAPoL-Start
    A->>S: EAP-Request/Identity
    S->>A: EAP-Response/Identity
    A->>R: RADIUS Access-Request (EAP)

    Note over S,R: Troca do método EAP (ex: EAP-TLS)
    R->>A: RADIUS Access-Challenge
    A->>S: EAP-Request (Método)
    S->>A: EAP-Response (Credenciais)
    A->>R: RADIUS Access-Request

    R->>D: Validação de credenciais/certificado
    D->>R: Resultado da validação

    alt Sucesso na Autenticação
        R->>A: RADIUS Access-Accept<br/>(Atributos de VLAN e ACL)
        A->>S: EAP-Success
        Note over S,A: Porta transita para o estado autorizado
    else Falha na Autenticação
        R->>A: RADIUS Access-Reject
        A->>S: EAP-Failure
        Note over S,A: Porta permanece não autorizada
    end
```

### Autenticação EAP-TEAP

```mermaid
sequenceDiagram
    participant S as Suplicante
    participant A as Autenticador
    participant R as Servidor RADIUS (Cisco ISE)
    participant D as Diretório / CA

    Note over S,A: Porta inicia no estado não autorizado

    %% Fase 1 - Estabelecimento do túnel TLS
    S->>A: EAPoL-Start
    A->>S: EAP-Request/Identity
    S->>A: EAP-Response/Identity
    A->>R: RADIUS Access-Request (EAP-TEAP)

    Note over S,R: Estabelecimento do túnel TLS (TEAP Outer Tunnel)
    R->>A: RADIUS Access-Challenge (TLS Handshake)
    A->>S: EAP-Request (TEAP/TLS)
    S->>A: EAP-Response (TLS Handshake)
    A->>R: RADIUS Access-Request (TEAP/TLS)

    R->>D: Validação do certificado do servidor
    D->>R: Resultado da validação

    %% Fase 2 - Autenticação interna (Inner Authentication)
    Note over S,R: Autenticação interna dentro do túnel TEAP

    S->>A: TEAP-Response (Credenciais de Máquina)
    A->>R: RADIUS Access-Request (Machine Auth)
    R->>D: Validação da identidade da máquina
    D->>R: Resultado da validação

    S->>A: TEAP-Response (Credenciais do Usuário)
    A->>R: RADIUS Access-Request (User Auth)
    R->>D: Validação da identidade do usuário
    D->>R: Resultado da validação

    %% Decisão de política
    alt Sucesso na Autenticação (Máquina + Usuário)
        R->>A: RADIUS Access-Accept<br/>(VLAN, dACL, SGT)
        A->>S: EAP-Success
        Note over S,A: Porta transita para o estado autorizado
    else Falha na Autenticação
        R->>A: RADIUS Access-Reject
        A->>S: EAP-Failure
        Note over S,A: Porta permanece não autorizada
    end
```

### MAC Authentication Bypass (MAB)

```mermaid
flowchart TD
    DEVICE[Dispositivo conecta] --> DOT1X{Compatível com<br/>802.1X?}

    DOT1X -->|Sim| AUTH[Autenticação 802.1X]
    DOT1X -->|Não/Timeout| MAB_START[Início do MAB]

    MAB_START --> MAC_CHECK{MAC presente<br/>na base de dados?}
    MAC_CHECK -->|Sim| ASSIGN[Atribuir VLAN de acordo com política]
    MAC_CHECK -->|Não| DENY[Acesso Negado]
```

### MAB Configuration Requirements

| Parâmetro                      | Configuração Recomendada                                                        | Justificativa Técnica                                                                          |
| ------------------------------ | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Timeout do MAB                 | Iniciar o MAB após o timeout do 802.1X (≈ 30 segundos)                          | Garantir prioridade ao 802.1X, utilizando MAB apenas como mecanismo de fallback                |
| Formato do endereço MAC        | Padronização em minúsculas (lowercase) e com separador (ex.: xx-xx-xx-xx-xx-xx) | Assegurar consistência no cadastro e na correlação de identidades no ISE                       |
| Atributo RADIUS utilizado      | Calling-Station-Id                                                              | Identificar o dispositivo com base no endereço MAC enviado pelo NAD (switch/AP)                |
| Device Profiling               | Habilitado e obrigatório                                                        | Validar o tipo de dispositivo e reduzir o risco de autenticação indevida baseada apenas no MAC |
| Intervalo de re-profiling      | 24 horas                                                                        | Detectar alterações de perfil e possíveis tentativas de spoofing de MAC                        |
| Política para MAC desconhecido | Bloqueio ou quarentena (VLAN restrita / ACL de contenção)                       | Aplicar o princípio de segurança por padrão (default deny)                                     |


### MAB Use Cases

MAB provides network access for devices that cannot perform 802.1X authentication:

| Device Category | Examples | MAB Policy |
|-----------------|----------|------------|
| Network printers | Enterprise print devices | Registered MAC, printer VLAN |
| Building systems | HVAC, access control, elevators | Registered MAC, IoT VLAN |
| Medical devices | Monitors, diagnostic equipment | Registered MAC, restricted VLAN |
| AV equipment | Displays, projectors | Registered MAC, AV VLAN |
| Legacy systems | Older equipment without supplicant | Registered MAC, legacy VLAN |




### Autenticação EAP-TEAP + MAB
```mermaid
sequenceDiagram
    participant S as Suplicante
    participant A as Autenticador (Switch/AP)
    participant R as Servidor RADIUS (Cisco ISE)
    participant D as Diretório / CA

    Note over S,A: Porta inicia no estado não autorizado

    %% Fase 1 - Tentativa TEAP (802.1X)
    S->>A: EAPoL-Start
    A->>S: EAP-Request/Identity
    S->>A: EAP-Response/Identity
    A->>R: RADIUS Access-Request (EAP-TEAP)

    Note over S,R: Estabelecimento do túnel TLS (TEAP)
    R->>A: RADIUS Access-Challenge (TEAP/TLS)
    A->>S: EAP-Request (TEAP/TLS)
    S->>A: EAP-Response (TEAP/TLS)
    A->>R: RADIUS Access-Request (TEAP/TLS)

    %% Autenticação interna TEAP
    R->>D: Validação de identidades (máquina/usuário)
    D->>R: Resultado da validação

    alt TEAP bem-sucedido
        R->>A: RADIUS Access-Accept<br/>(VLAN, dACL, SGT)
        A->>S: EAP-Success
        Note over S,A: Porta autorizada via TEAP
    else TEAP falhou ou não suportado
        Note over S,A: Fallback para MAB

        %% Fase 2 - MAB
        A->>R: RADIUS Access-Request (MAC Address)
        R->>D: Consulta de MAC / Profiling
        D->>R: Resultado da validação

        alt MAB autorizado
            R->>A: RADIUS Access-Accept<br/>(Política restrita)
            Note over S,A: Porta autorizada via MAB
        else MAB rejeitado
            R->>A: RADIUS Access-Reject
            Note over S,A: Porta permanece não autorizada
        end
    end
```
