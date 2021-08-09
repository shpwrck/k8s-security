sequenceDiagram
    participant U as User
    participant I as Identity Provider
    participant K as Kubectl
    participant A as API Server
    participant Z as Authorization Server
    U->>I: Login to IdP
    I-->>U: Provide Token
    U->>K: Call Kubectl with Token
    K->>A: Forward Token
    A->>A: Is JWT valid?
    A->>A: Has the JWT expired?
    A->>Z: Send Subject Access Review
    loop 
        Z->>Z: Evaluate Subject Access Review Against Policies
    end
    A-->>K: Authorized (Result and Report)
    K-->>U: Report
