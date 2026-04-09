flowchart LR
    subgraph Providers[Email Providers]
        G2[Gmail]
        O2[Outlook]
        Y2[Yahoo]
        X2[Other Accounts]
    end

    subgraph Server["Cloud/VPS or Self-Controlled Server (Ubuntu)"]
        S1["Mail Ingest<br/>(fetchmail / getmail / offlineimap)"]
        S2["Primary Copy #1<br/>(Central Maildir Archive)"]
        S3["IMAP Service<br/>(Dovecot)"]
    end

    subgraph Local["Local Server"]
        S4["Backup Copy #2<br/>(rsnapshot repository)"]
    end

    subgraph Client[Your Devices]
        C2["Mail Client(s)"]
    end

    subgraph Offsite[Offsite Protection]
        B2["Offsite Copy #3<br/>(Backblaze B2, Encrypted)"]
        R2["Additional Offsite Copy<br/>(Cloudflare R2, Encrypted)"]
    end

    G2 -->|IMAP/POP pull| S1
    O2 -->|IMAP/POP pull| S1
    Y2 -->|IMAP/POP pull| S1
    X2 -->|IMAP/POP pull| S1
    S1 --> S2
    S2 --> S3
    S3 --> C2
    S2 -->|Encrypted sync via tailnet VPN| S4
    S2 --> B2
    S2 --> R2

    Q{{Deletion worker}}
    S4 --> Q
    B2 --> Q
    R2 --> Q
    Q -->|Delete provider copy only after retention + backup checks| Providers
