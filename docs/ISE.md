# ISE

### Deployment Redundânte (VM)

```mermaid
graph TB
    %% =========================
    %% Prédios / Sites de Rede
    %% =========================
    subgraph PREDIOS["Prédios / Sites de Rede"]
        
        subgraph PREDIO_A["Prédio A"]
            SW_A1["Switch A1"]
            SW_A2["Switch A2"]
            AP_A["Access Points"]
        end

        subgraph PREDIO_B["Prédio B"]
            SW_B1["Switch B1"]
            AP_B["Access Points"]
        end

    end

    %% =========================
    %% Infraestrutura ISE
    %% =========================
    subgraph ISE_INFRA["Infraestrutura Cisco ISE"]
        
        subgraph HOST_1["Host Físico 1"]
            ISE_1["ISE Nó 1 (VM)"]
        end

        subgraph HOST_2["Host Físico 2"]
            ISE_2["ISE Nó 2 (VM)"]
        end

    end

    %% =========================
    %% Serviços de Backend
    %% =========================
    subgraph BACKEND["Serviços de Backend"]
        AD["Active Directory (AD)"]
        CA["Autoridade Certificadora (CA)"]
    end

    %% =========================
    %% Fluxo RADIUS (802.1X / MAB)
    %% =========================
    SW_A1 & SW_A2 & AP_A -->|RADIUS Primário| ISE_1
    SW_A1 & SW_A2 & AP_A -.->|RADIUS Secundário| ISE_2

    SW_B1 & AP_B -->|RADIUS Primário| ISE_2
    SW_B1 & AP_B -.->|RADIUS Secundário| ISE_1

    %% =========================
    %% Comunicação com Backend
    %% =========================
    ISE_1 & ISE_2 --> AD
    ISE_1 & ISE_2 --> CA
```

### Atributos RADIUS para Aplicação de Políticas

| Attribute | Number | Purpose | Example |
|-----------|--------|---------|---------|
| Tunnel-Type | 64 | VLAN assignment | VLAN |
| Tunnel-Medium-Type | 65 | Medium type | IEEE-802 |
| Tunnel-Private-Group-ID | 81 | VLAN ID/name | 20 |
| Filter-Id | 11 | ACL assignment | CORP-ACL |
| Session-Timeout | 27 | Reauth interval | 28800 (8 hours) |
| Termination-Action | 29 | Post-session action | RADIUS-Request |
