# Chess Hive — App Workflow

## 1. Entry & Auth

```mermaid
flowchart TD
    A([Launch]) --> B{Session?}
    B -->|No| C[Login]
    B -->|Guest| H[Home]
    B -->|Yes| G{Profile?}
    C --> D[Register]
    D --> E[Role]
    E --> F[Complete Profile]
    F --> P{Approval?}
    G -->|Incomplete| F
    G -->|Pending| P
    G -->|OK| H
    P -->|Wait| W[Pending Approval]
    P -->|Approved| H
    C -.Skip.-> H
```

## 2. Main Shell (Bottom Nav)

```mermaid
flowchart LR
    H[🏠 Home] --- T[🏆 Tournaments]
    T --- M[🛒 Marketplace]
    M --- C[👥 Community]
    C --- X[💬 Chat]
    X --- P[👤 Profile]
```

## 3. Tournaments

```mermaid
flowchart TD
    T[Tournaments] --> L[List]
    L --> D[Detail]
    D --> R[Register]
    D --> S[Standings]
    D --> E[Enter Results]
    L --> N[Create New]
```

## 4. Marketplace

```mermaid
flowchart TD
    M[Marketplace] --> L[Listings]
    L --> D[Detail]
    L --> A[Sell → Apply]
    A --> C[Create Listing]
```

## 5. Club Membership

```mermaid
flowchart TD
    U[User on Club Detail] -->|Apply to Join| AP[clubId=X<br/>status=pending]
    H[Club Head] -.Invite.-> AP
    AP --> P[Pending List]
    P --> R{Head Reviews}
    R -->|Approve| AC[status=active<br/>✅ Member]
    R -->|Reject| RJ[clubId=null<br/>status=active]
    AP -.User cancels.-> RJ
    AC -.User leaves.-> RJ
```

Stored on `users/{uid}` → `clubId` + `status`. Either the user applies or the head invites; only the head/admin can flip `pending → active`.

## 6. Community

```mermaid
flowchart TD
    C[Community] --> CL[Clubs]
    C --> PL[Players]
    C --> CO[Coaches]
    C --> AR[Articles]
    CL --> CD[Club Detail]
    CL --> RC[Register Club]
    CO --> CDT[Coach Detail]
    AR --> AD[Article Detail]
    AR --> WR[Write Article]
```

## 7. Chat

```mermaid
flowchart LR
    X[Conversations] --> TH[Thread]
    TH --> SE[Send Message]
```

## 8. Profile

```mermaid
flowchart TD
    P[Profile] --> E[Edit]
    P --> S[Settings]
    P --> AD[Admin]
    P --> N[🔔 Notifications]
    P --> SR[🔍 Search]
```

## 9. Roles & Gates

```mermaid
flowchart LR
    G[Guest] -->|Sign in| U[User]
    U -->|Apply| SE[Seller]
    U -->|Register| OR[Organizer]
    U -->|Apply| CO[Coach]
    U --> AD[Admin]
    AD -->|Approves| SE
    AD -->|Approves| OR
    AD -->|Approves| CO
```
