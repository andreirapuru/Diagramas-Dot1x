## Arquitetura 802.1X

### Componentes
```mermaid
graph LR
    subgraph ENDPOINT["Camada de Endpoint"]
        SUP["Cliente 802.1X (Supplicant)<br/>Dispositivos corporativos"]
        IOT["Dispositivos sem 802.1X<br/>(IoT / Impressoras / Telefones IP)"]
        GUEST["Dispositivos de Convidados"]
    end

    subgraph ACCESS["Camada de Acesso à Rede"]
        AUTH["Dispositivo de Acesso (Authenticator)<br/>NADs - Switch / Access Point"]
    end

    subgraph IDENTITY["Camada de Identidade e Controle de Acesso"]
        ISE["Cisco Identity Services Engine (ISE)<br/>RADIUS / Policy Engine Server"]
        AD["Active Directory / LDAP"]
        PKI["Infraestrutura PKI / Autoridade Certificadora"]
    end

    %% Fluxos de autenticação
    SUP <-->|"802.1X / EAPoL"| AUTH
    IOT <-->|"MAB (MAC Authentication Bypass)"| AUTH
    GUEST <-->|"Acesso Web / Captive Portal"| AUTH

    AUTH <-->|"RADIUS (UDP 1812/1813)"| ISE

    %% Integrações do ISE
    ISE <--> AD
    ISE <--> PKI
```

### Estado da porta

```mermaid
stateDiagram-v2
    [*] --> Não_Autorizado: Link da porta ativo

    Não_Autorizado --> Autenticando: EAPoL-Start recebido

    Autenticando --> Autorizado: Access-Accept (802.1X)
    Autenticando --> Não_Autorizado: Access-Reject (802.1X)
    Autenticando --> MAB: Timeout (sem EAPoL)

    MAB --> Autorizado: Access-Accept (MAB)
    MAB --> Não_Autorizado: Access-Reject (MAB)

    Autorizado --> Não_Autorizado: Logoff / Link down
    Autorizado --> Reautenticando: Timer de reautenticação

    Reautenticando --> Autorizado: Access-Accept
    Reautenticando --> Não_Autorizado: Access-Reject

    note right of Não_Autorizado
        Nenhum tráfego permitido
        exceto quadros EAPoL
    end note

    note right of Autorizado
        Acesso à rede
        conforme atributos RADIUS
    end note

    note right of MAB
        MAC Authentication Bypass
        acionado quando o 802.1X não tem retorno
    end note
```

### Escolha do Método de Autenticação

```mermaid
flowchart TD
    START[Avaliação do Tipo de Dispositivo] --> Q1{O dispositivo suporta<br/>certificados?}

    Q1 -->|Sim| Q2{Gerenciado por<br/>MDM/PKI?}
    Q1 -->|Não| MAB["MAC Authentication Bypass (MAB)<br/>(Com segmentação)"]

    Q2 -->|Sim| EAPTLS["EAP-TLS<br/>(Único permitido)"]
    Q2 -->|Não| ENROLL["Registrar no PKI/MDM<br/>antes da implantação"]

    ENROLL --> EAPTLS

    EAPTLS --> RESULT[Aplicar política de Acesso/VLAN/ACL/TAG]
    MAB --> RESULT
```

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

## Arquitetura ISE / Redundância

### Redundant Deployment

```mermaid
graph TB
    subgraph SITES["Network Sites"]
        subgraph SITE_A["Site A"]
            SW_A1["Switch A1"]
            SW_A2["Switch A2"]
            AP_A["Access Points"]
        end

        subgraph SITE_B["Site B"]
            SW_B1["Switch B1"]
            AP_B["Access Points"]
        end
    end

    subgraph RADIUS_INFRA["RADIUS Infrastructure"]
        RAD_P["Primary RADIUS<br/>(Datacenter A)"]
        RAD_S["Secondary RADIUS<br/>(Datacenter B)"]
    end

    subgraph BACKEND["Backend Services"]
        AD["Directory Service"]
        CA["Certificate Authority"]
        NPS["Network Policy Service"]
    end

    SW_A1 & SW_A2 & AP_A -->|Primary| RAD_P
    SW_A1 & SW_A2 & AP_A -.->|Failover| RAD_S
    SW_B1 & AP_B -->|Primary| RAD_P
    SW_B1 & AP_B -.->|Failover| RAD_S

    RAD_P & RAD_S --> AD
    RAD_P & RAD_S --> CA
    RAD_P & RAD_S --> NPS
```

### RADIUS Attributes for Policy Enforcement

| Attribute | Number | Purpose | Example |
|-----------|--------|---------|---------|
| Tunnel-Type | 64 | VLAN assignment | VLAN |
| Tunnel-Medium-Type | 65 | Medium type | IEEE-802 |
| Tunnel-Private-Group-ID | 81 | VLAN ID/name | 20 |
| Filter-Id | 11 | ACL assignment | CORP-ACL |
| Session-Timeout | 27 | Reauth interval | 28800 (8 hours) |
| Termination-Action | 29 | Post-session action | RADIUS-Request |

## Wired 802.1X Implementation

### Switch Port Configuration Standards

```mermaid
graph TD
    subgraph PORT_TYPES["Port Authentication Modes"]
        SINGLE["Single-Host Mode<br/>One MAC per port"]
        MULTI_AUTH["Multi-Auth Mode<br/>Multiple MACs, each authenticated"]
        MULTI_DOMAIN["Multi-Domain Mode<br/>Voice + Data VLANs"]
        MULTI_HOST["Multi-Host Mode<br/>One auth, all allowed"]
    end

    SINGLE --> USE1["Single workstation ports"]
    MULTI_AUTH --> USE2["Conference rooms<br/>with hubs"]
    MULTI_DOMAIN --> USE3["IP phones with<br/>passthrough PC"]
    MULTI_HOST --> USE4["Not recommended<br/>(Security risk)"]
```

### Port Configuration Requirements

| Setting | Standard Value | Rationale |
|---------|---------------|-----------|
| Authentication mode | Multi-Domain (voice+data) or Multi-Auth | Support IP phones and conferencing |
| Host mode | Multi-auth preferred | Per-device authentication |
| Periodic reauthentication | Enabled, 8 hours | Session validation |
| Quiet period | 60 seconds | Retry delay after failure |
| Tx period | 30 seconds | EAP request interval |
| Supplicant timeout | 30 seconds | Client response timeout |
| Server timeout | 30 seconds | RADIUS response timeout |
| Maximum requests | 3 | Retry attempts |
| Guest VLAN | Enabled for designated ports | Unauthenticated access where required |
| Auth-fail VLAN | Enabled | Quarantine for failed auth |
| Critical VLAN | Enabled | Access when RADIUS unavailable |

### Port Exception Categories

```mermaid
flowchart TD
    subgraph EXCEPTIONS["802.1X Port Exceptions"]
        INFRA["Infrastructure Ports<br/>- Uplinks to other switches<br/>- Router connections<br/>- Server ports (authenticated elsewhere)"]

        DEDICATED["Dedicated Device Ports<br/>- Printers (MAB)<br/>- Building systems (MAB)<br/>- Legacy devices (MAB)"]

        PUBLIC["Public Access Ports<br/>- Lobby kiosks<br/>- Public terminals<br/>(Separate VLAN, filtered)"]
    end

    INFRA --> TRUST["Trusted port<br/>(No 802.1X)"]
    DEDICATED --> MAB_AUTH["MAB with device<br/>registration"]
    PUBLIC --> GUEST["Guest VLAN<br/>(Internet only)"]
```



## Wireless 802.1X Integration

### Wireless-Specific Considerations

| Aspect | Wired | Wireless | Implication |
|--------|-------|----------|-------------|
| Physical port | One device per port | Multiple clients per AP | Use multi-auth mode on AP |
| Roaming | N/A | Client moves between APs | Fast BSS transition (802.11r) |
| Key management | EAPoL-Key | 4-way handshake + PMK caching | OKC/802.11r for fast roaming |
| Encryption | Optional (MACsec) | Required (WPA3) | Always encrypt wireless |

### Fast Roaming Support

```mermaid
sequenceDiagram
    participant C as Client
    participant AP1 as Current AP
    participant AP2 as Target AP
    participant R as RADIUS

    Note over C,AP1: Initial authentication
    C->>AP1: Full 802.1X + 4-way handshake
    AP1->>R: Full RADIUS exchange
    R->>AP1: PMK generated

    Note over C,AP2: Client roams (802.11r FT)
    C->>AP2: FT Authentication Request<br/>(includes PMKR1)
    AP2->>AP1: PMK-R1 request (over DS)
    AP1->>AP2: PMK-R1 response
    AP2->>C: FT Authentication Response
    C->>AP2: Reassociation Request
    AP2->>C: Reassociation Response

    Note over C,AP2: Roam complete<br/>(<50ms typical)
```



## Security Considerations

### Threat Mitigation

| Threat | Without 802.1X | With 802.1X |
|--------|----------------|-------------|
| Unauthorized device connection | Possible | Blocked at port |
| MAC spoofing | No detection | Limited by profiling |
| Rogue access points | Not controlled | Detected/blocked |
| Lateral movement | Unrestricted | VLAN isolation |
| Credential theft | Network-wide impact | Limited to authorized resources |
| Physical port abuse | Any device connects | Authentication required |

### MACsec Integration (IEEE 802.1AE)

For high-security environments, 802.1X can enable MACsec encryption:

```mermaid
flowchart LR
    subgraph STANDARD["Standard 802.1X"]
        S_AUTH["Authentication"]
        S_DATA["Unencrypted<br/>Layer 2 traffic"]
    end

    subgraph MACSEC["802.1X + MACsec"]
        M_AUTH["Authentication +<br/>Key Agreement"]
        M_DATA["Encrypted<br/>Layer 2 traffic"]
    end

    S_AUTH --> S_DATA
    M_AUTH --> M_DATA

    S_DATA -->|"Vulnerable to<br/>sniffing"| RISK["Risk"]
    M_DATA -->|"Wire-speed<br/>encryption"| SECURE["Secure"]
```

## NIST Alignment

### NIST SP 800-53 Control Mapping

| Control ID | Control Name | 802.1X Implementation |
|------------|--------------|----------------------|
| AC-3 | Access Enforcement | Port-based access control |
| AC-17 | Remote Access | VPN integration with 802.1X |
| AU-2 | Audit Events | RADIUS accounting logs |
| AU-3 | Content of Audit Records | Authentication success/failure details |
| IA-2 | Identification and Authentication | EAP-TLS certificates |
| IA-3 | Device Identification and Authentication | Device certificates, MAB |
| IA-5 | Authenticator Management | Certificate lifecycle |
| IA-8 | Identification and Authentication (Non-Org Users) | Guest VLAN policies |
| SC-8 | Transmission Confidentiality | MACsec option |
| SC-23 | Session Authenticity | EAP session binding |

### NIST SP 800-63B-4 Alignment

| Assurance Level | Authentication Method | 802.1X Equivalent | Status |
|-----------------|----------------------|-------------------|--------|
| AAL1 | Single-factor | Username/password | ❌ **Forbidden** |
| AAL2 | Multi-factor | EAP-TLS with device certificate | ✅ Minimum required |
| AAL3 | Hardware crypto | EAP-TLS with TPM-backed certificate | ✅ Recommended |

> **Policy:** This standard requires AAL2 minimum (EAP-TLS with device certificates). AAL3 (TPM-backed certificates) is recommended for high-security environments.

## Guia para Resolução de ProblemasTroubleshooting Guide

| Sintoma                     | Causa Provável                        | Resolução                                                                  |
| --------------------------- | ------------------------------------- | -------------------------------------------------------------------------- |
| Timeout de autenticação     | Suplicante não respondendo            | Verificar se o suplicante está habilitado e se o SSID/porta estão corretos |
| Erro de certificado         | Certificado expirado ou não confiável | Verificar a cadeia de certificados e datas de validade                     |
| Timeout de RADIUS           | Servidor inacessível                  | Verificar conectividade e shared secret                                    |
| Falha na atribuição de VLAN | Atributos RADIUS ausentes             | Configurar os atributos corretos no servidor                               |
| Falhas intermitentes        | Reautenticação durante a sessão       | Aumentar o tempo de sessão                                                 |
| Falha em dispositivos MAB   | MAC não registrado                    | Adicionar o MAC à base de endereços autorizados                            |


### Diagnostic Flow

```mermaid
flowchart TD
    ISSUE[Falha de autenticação] --> CHECK1{Quadros EAPoL<br/>chegam ao switch?}
    CHECK1 -->|Não| FIX1[Verificar suplicante,<br/>configuração da porta]
    CHECK1 -->|Sim| CHECK2{Servidor RADIUS<br/>acessível?}
    CHECK2 -->|Não| FIX2[Verificar roteamento,<br/>regras de firewall]
    CHECK2 -->|Sim| CHECK3{Shared secret<br/>correto?}
    CHECK3 -->|Não| FIX3[Atualizar shared secret]
    CHECK3 -->|Sim| CHECK4{Certificado<br/>válido?}
    CHECK4 -->|Não| FIX4[Renovar certificado,<br/>verificar cadeia]
    CHECK4 -->|Sim| CHECK5{Política permite<br/>o acesso?}
    CHECK5 -->|Não| FIX5[Ajustar política RADIUS]
    CHECK5 -->|Sim| ESCALATE[Escalonar para<br/>suporte do fabricante]
```

