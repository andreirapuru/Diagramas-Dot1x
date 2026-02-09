# Diagramas-Dot1x

```mermaid
graph LR
    subgraph ENDPOINT["Endpoint Layer"]
        SUP["Supplicant<br/>(Client software)"]
    end

    subgraph NETWORK["Network Layer"]
        AUTH["Authenticator<br/>(Switch/AP)"]
    end

    subgraph BACKEND["Backend Layer"]
        RADIUS["Authentication Server<br/>(RADIUS)"]
        AD["Directory Service"]
        CA["Certificate Authority"]
        NAC["NAC Policy Engine<br/>(Optional)"]
    end

    SUP <-->|"EAP over LAN<br/>(EAPoL)"| AUTH
    AUTH <-->|"RADIUS<br/>(UDP 1812/1813)"| RADIUS
    RADIUS <--> AD
    RADIUS <--> CA
    RADIUS <--> NAC
