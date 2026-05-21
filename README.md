# ⚖️ LAWL - Detailed Workflow Diagrams

This document provides a highly detailed walkthrough of the core workflows in the **Lawl** mobile application ecosystem. It highlights how the Flutter frontend (using GetX state management), Node.js backend, Socket.IO real-time server, native Android platform channels, and third-party integrations (100ms, Razorpay, FCM, eCourts) work together.

---

## 1. Master System Architecture Flowchart
The following diagram maps the entire system components, showing the interactions between the client applications, backend, socket servers, and third-party integrations.

```mermaid
graph TD
    %% Styling
    classDef client fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0f172a;
    classDef server fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#0f172a;
    classDef external fill:#f3e8ff,stroke:#7c3aed,stroke-width:2px,color:#0f172a;

    subgraph Clients ["Flutter Mobile Apps (GetX State)"]
        ClientApp["Client App UI & Controllers"]
        LawyerApp["Lawyer App UI & Controllers"]
        PlatformChannel["Platform Channel (MethodChannel)"]
        NativeRingtone["Android RingtoneManager / iOS Audio"]
    end

    subgraph BackendServices ["Backend & Realtime Services"]
        APIServer["Node.js API Server (REST)"]
        SocketServer["Socket.IO Server (Signaling)"]
        Database[("MongoDB / Postgres Database")]
    end

    subgraph ThirdParty ["Third-Party Service Providers"]
        FCM["Firebase Cloud Messaging (FCM)"]
        HMS["100ms SDK (WebRTC Rooms)"]
        Razorpay["Razorpay Payment Gateway"]
        eCourtsScraper["eCourts Portal / Scraper"]
    end

    %% Client Relations
    ClientApp <--> APIServer
    ClientApp <--> SocketServer
    LawyerApp <--> APIServer
    LawyerApp <--> SocketServer

    %% Native Channels
    LawyerApp <--> PlatformChannel
    PlatformChannel <--> NativeRingtone

    %% Backend Relations
    APIServer <--> Database
    SocketServer <--> Database
    APIServer -.-> FCM
    
    %% Third Party Connections
    ClientApp <--> Razorpay
    ClientApp <--> HMS
    LawyerApp <--> HMS
    APIServer <--> HMS
    APIServer <--> eCourtsScraper

    %% Apply Styling Classes
    class ClientApp,LawyerApp,PlatformChannel,NativeRingtone client;
    class APIServer,SocketServer,Database server;
    class FCM,HMS,Razorpay,eCourtsScraper external;
```

---

## 2. Authentication & Onboarding Workflow
This flowchart outlines the process of user registration, role selection, and the lawyer verification pipeline (KYC approval/rejection).

```mermaid
flowchart TD
    %% Styling
    classDef step fill:#f8fafc,stroke:#64748b,stroke-width:1px,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#0f172a;
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d;
    classDef failure fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d;

    Start(["Start Onboarding / Login"]) --> InputPhone["Input Mobile Number"]
    InputPhone --> ReqOTP["Request OTP (API Call)"]
    ReqOTP --> SendSMS["SMS Gateway Dispatches OTP"]
    SendSMS --> EnterOTP["User Enters OTP"]
    EnterOTP --> VerifyOTP{"OTP Valid?"}

    VerifyOTP -- No --> AlertError["Display Error Toast"]
    AlertError --> EnterOTP

    VerifyOTP -- Yes --> CheckProfile{"Profile Exists?"}

    CheckProfile -- Yes --> CheckRole{"Role Type?"}
    
    CheckProfile -- No --> FillDetails["Fill Basic Details (Name, Email, City)"]
    FillDetails --> ChooseRole{"Select Role"}
    
    ChooseRole -- "Client User" --> SaveClient["Create Client Profile"]
    SaveClient --> ClientDashboard(["Access Client Dashboard"])
    
    ChooseRole -- "Lawyer / Advocate" --> SaveLawyer["Create Basic Lawyer Profile"]
    
    CheckRole -- "Client User" --> ClientDashboard
    CheckRole -- "Lawyer / Advocate" --> CheckVerificationStatus
    
    SaveLawyer --> UploadKYC["Upload KYC Docs (Bar Council ID, Registration Certificate)"]
    UploadKYC --> CheckVerificationStatus{"Verification Status"}
    
    CheckVerificationStatus -- "Approved" --> LawyerDashboard(["Access Lawyer Dashboard"])
    CheckVerificationStatus -- "Pending" --> BlockScreen["Show Pending Verification Overlay"]
    CheckVerificationStatus -- "Rejected" --> ReUploadKYC["Show Rejection Reason & Re-upload Request"]
    
    BlockScreen --> ExitOrWait["Wait for Admin Approval"]
    ReUploadKYC --> UploadKYC

    class InputPhone,ReqOTP,SendSMS,EnterOTP,AlertError,FillDetails,SaveClient,SaveLawyer,UploadKYC,BlockScreen,ReUploadKYC,ExitOrWait step;
    class VerifyOTP,CheckProfile,CheckRole,ChooseRole,CheckVerificationStatus decision;
    class ClientDashboard,LawyerDashboard success;
```

---

## 3. Session Lifecycle & Socket Signaling
Consultations in LAWL are structured around a **Session-First** flow, where chat/call features are only enabled inside an active session.

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client User
    participant Socket as Socket.IO Server
    participant DB as Backend Database
    actor Lawyer as Lawyer User

    %% Request Initiation
    Client->>Socket: emit("request_session", { lawyerId, sessionType })
    activate Socket
    Socket->>DB: Create Session (status: "requested")
    Socket-->>Lawyer: emit("new_session_request", sessionPayload)
    deactivate Socket
    Note over Lawyer: Lawyer receives request list in real-time

    %% Accept or Decline decision
    alt Lawyer accepts session request
        Lawyer->>Socket: emit("accept_session", { sessionId })
        activate Socket
        Socket->>DB: Update Session (status: "active")
        Socket-->>Client: emit("session_active", sessionPayload)
        Socket-->>Lawyer: emit("session_active", sessionPayload)
        deactivate Socket
        Note over Client, Lawyer: Chat screen unlocked, Call features active
    else Lawyer rejects session request
        Lawyer->>Socket: emit("reject_session", { sessionId })
        activate Socket
        Socket->>DB: Update Session (status: "rejected")
        Socket-->>Client: emit("session_rejected", { sessionId })
        deactivate Socket
        Note over Client: Notified of rejection, refund credited
    else Client cancels request before response
        Client->>Socket: emit("cancel_session", { sessionId })
        activate Socket
        Socket->>DB: Update Session (status: "cancelled")
        Socket-->>Lawyer: emit("session_cancelled", { sessionId })
        deactivate Socket
    end

    %% Session Ending
    Note over Client, Lawyer: Active consultation in progress...
    alt User/Lawyer ends session manually
        Client->>Socket: emit("end_session", { sessionId })
        activate Socket
        Socket->>DB: Update Session (status: "ended")
        Socket-->>Lawyer: emit("session_ended", { sessionId })
        deactivate Socket
        Note over Client, Lawyer: Navigate to feedback / ratings screen
    end
```

---

## 4. Call Lifecycle & Native Ringtone Flow
This flow details how real-time call requests invoke native ringtone loop mechanisms in both foreground and background states, transitioning into WebRTC video/voice sessions via the 100ms SDK.

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client User
    participant Socket as Socket.IO Server
    participant FCM as Firebase Push Server
    participant LawyerCtrl as Lawyer CallController (GetX)
    participant LocalNotif as LocalNotificationService
    participant Native as Native Android (MainActivity)
    participant HMS as 100ms SDK

    Client->>Socket: emit("request_call", { sessionId, callType })
    
    %% Foreground vs Background Branching
    Note over Socket, LawyerCtrl: Scenario A: Lawyer App is in Foreground
    Socket-->>LawyerCtrl: emit("call_incoming", callPayload)
    
    LawyerCtrl->>LawyerCtrl: Set internal call state (ringing)
    LawyerCtrl->>Native: MethodChannel("playRingtone")
    activate Native
    Native->>Native: RingtoneManager.getRingtone(getDefaultUri(TYPE_RINGTONE))
    Native->>Native: Start looping default system ringtone
    deactivate Native
    LawyerCtrl->>LawyerCtrl: Navigate to /incoming-call (FullScreen Overlay)

    Note over Socket, LocalNotif: Scenario B: Lawyer App is in Background / Killed
    Socket->>FCM: Trigger Push Notification (INCOMING_CALL payload)
    FCM-->>LocalNotif: Receive High-Priority FCM Push Message
    
    LocalNotif->>LocalNotif: parseMessageData()
    LocalNotif->>Native: showCallNotification() with fullScreenIntent
    activate Native
    Native->>Native: Play system channel default ringtone
    Native->>Native: Display heads-up incoming call alert banner
    deactivate Native

    %% Answering Flow
    Note over LawyerCtrl: Lawyer Decides Action
    
    alt Lawyer Answers Call
        LawyerCtrl->>Native: MethodChannel("stopRingtone")
        activate Native
        Native->>Native: ringtone.stop() & release resource
        deactivate Native
        
        LawyerCtrl->>Socket: emit("accept_call", { sessionId })
        Socket-->>Client: emit("call_accepted", callPayload)
        
        %% WebRTC Join Room
        par Client Joins Room
            Client->>HMS: Request 100ms token & Join room
        and Lawyer Joins Room
            LawyerCtrl->>HMS: Request 100ms token & Join room
        end
        
        Note over Client, HMS: WebRTC Call established (Video/Voice Active)

    else Lawyer Declines Call
        LawyerCtrl->>Native: MethodChannel("stopRingtone")
        activate Native
        Native->>Native: ringtone.stop()
        deactivate Native
        LawyerCtrl->>Socket: emit("decline_call", { sessionId })
        Socket-->>Client: emit("call_declined", { sessionId })
        LawyerCtrl->>LawyerCtrl: Navigate back / Close call screen

    else Call Timeout (No Answer for 30s)
        Socket-->>Client: emit("call_timeout", { sessionId })
        Socket-->>LawyerCtrl: emit("call_cancelled", { sessionId })
        LawyerCtrl->>Native: MethodChannel("stopRingtone")
        activate Native
        Native->>Native: ringtone.stop()
        deactivate Native
        LawyerCtrl->>LawyerCtrl: Close call screen & show missed call
    end
```

---

## 5. Wallet, Billing, & Auto-Extension Flow
To maintain active sessions, the system monitors wallet balance, alerts the user, and automates extensions using Razorpay integration.

```mermaid
flowchart TD
    %% Styling
    classDef state fill:#f8fafc,stroke:#64748b,stroke-width:1px,color:#0f172a;
    classDef event fill:#f1f5f9,stroke:#3b82f6,stroke-width:1.5px,color:#1e3a8a;
    classDef decision fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#0f172a;
    classDef action fill:#dcfce7,stroke:#16a34a,stroke-width:1.5px,color:#14532d;
    classDef danger fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d;

    ActiveSession(["Session is Active (Timer Counting Down)"]) --> TimerCheck{"Time Remaining <= 2 Mins?"}
    
    TimerCheck -- No --> ActiveSession
    
    TimerCheck -- Yes --> EmitWarning["Socket Server Emits 'session_warning'"]
    
    EmitWarning --> ShowBottomSheet["Flutter UI: Display Extension Bottom Sheet"]
    
    ShowBottomSheet --> CheckBalance{"Client Wallet Balance >= Block Price?"}
    
    CheckBalance -- Yes --> ConfirmExtend["Client Taps 'Confirm Extension (+15m)'"]
    ConfirmExtend --> EmitExtend["Socket Emit 'extend_session'"]
    EmitExtend --> DBUpdate["Server Deducts Balance & Adds Time in DB"]
    DBUpdate --> SuccessAck["Socket Emits 'extension_success'"]
    SuccessAck --> ExtendCountdown["Flutter Updates Timer UI (+15 Mins)"]
    ExtendCountdown --> ActiveSession

    CheckBalance -- No --> PromptTopUp["Flutter UI Shows Top-Up Required State"]
    
    PromptTopUp --> SelectAmount["User Selects Top-Up Amount"]
    SelectAmount --> InitRazorpay["Initiate Razorpay SDK Checkout"]
    
    InitRazorpay --> RazorpayProcess{"Payment Success?"}
    
    RazorpayProcess -- No --> PaymentFailed["Display Payment Error Toast"]
    PaymentFailed --> ShowBottomSheet
    
    RazorpayProcess -- Yes --> RazorpayCallback["Razorpay SDK Returns Payment ID"]
    RazorpayCallback --> VerifyPayment["API Call: Verify Payment & Auto-Extend"]
    VerifyPayment --> APIDBUpdate["Server Validates Signature, Credits Wallet, Extends Session"]
    APIDBUpdate --> APISuccessAck["API Returns Success & Socket Emits Extension Success"]
    APISuccessAck --> ExtendCountdown

    TimerCheck -- Expiry reached (0s) --> EndSession["Socket Server Emits 'session_ended'"]
    EndSession --> TerminateTracks["100ms Audio/Video Tracks Terminated"]
    TerminateTracks --> GoToFeedback(["Close Session & Navigate to Ratings Page"])

    class ActiveSession,EmitWarning,ShowBottomSheet,ConfirmExtend,EmitExtend,DBUpdate,SuccessAck,ExtendCountdown,PromptTopUp,SelectAmount,InitRazorpay,RazorpayCallback,VerifyPayment,APIDBUpdate,APISuccessAck,PaymentFailed,EndSession,TerminateTracks state;
    class TimerCheck,CheckBalance,RazorpayProcess decision;
    class GoToFeedback action;
```

---

## 6. eCourts Scraper & Case Management Flow
This diagram details how advocates can search and add cases to their dashboard by either scanning a physical QR code or performing a query against state/district filters, scraper logic, and multi-result picking.

```mermaid
flowchart TD
    %% Styling
    classDef source fill:#f8fafc,stroke:#475569,stroke-width:1.5px,color:#0f172a;
    classDef process fill:#eff6ff,stroke:#2563eb,stroke-width:1.5px,color:#1e3a8a;
    classDef decision fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#0f172a;
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d;
    classDef error fill:#fee2e2,stroke:#dc2626,stroke-width:1.5px,color:#7f1d1d;

    StartCaseAdd(["Lawyer Dashboard: Add eCourts Case"]) --> ChooseMethod{"Choose Addition Method"}
    
    ChooseMethod -- "Scan QR Code" --> OpenScanner["Launch Mobile Scanner SDK"]
    OpenScanner --> ScanQR["Scan Physical Case Sheet QR Code"]
    ScanQR --> ExtractCNR["Extract CNR Number String"]
    ExtractCNR --> APIRequestCNR["API Call: /ecourts/scan-cnr"]
    
    ChooseMethod -- "Manual Filter Search" --> SelectLocation["Select State & District"]
    SelectLocation --> SelectCourt["Select Court Complex"]
    SelectCourt --> EnterCaseDetails["Enter Case Type, Case Number, & Year"]
    EnterCaseDetails --> APIRequestManual["API Call: /ecourts/search-number"]

    APIRequestCNR --> BackendScrape["Backend Scraper Scrapes eCourts Portal"]
    APIRequestManual --> BackendScrape
    
    BackendScrape --> ScrapeResult{"Parse Scrape Results"}
    
    ScrapeResult -- "Error / Not Found" --> ErrorState["Return Scrape Failed Event"]
    ErrorState --> ShowErrorToast["Flutter UI: Display Error Alert Banner"]
    ShowErrorToast --> StartCaseAdd

    ScrapeResult -- "Multiple Cases Match" --> ReturnMultiList["Return Matching Cases Index List"]
    ReturnMultiList --> DisplayPicker["Flutter UI: Display Selection Picker Sheet"]
    DisplayPicker --> UserSelectsCase["Lawyer Selects Exact Target Case"]
    UserSelectsCase --> APIRequestSelect["API Call: /ecourts/select-from-multi"]
    APIRequestSelect --> FetchFullDetails["Backend Scrapes Selected Case Details"]
    FetchFullDetails --> SaveToDB

    ScrapeResult -- "Single Direct Match" --> SaveToDB["Backend Saves Case Model to Database"]
    
    SaveToDB --> ReturnCaseModel["Return Structured CaseModel JSON"]
    ReturnCaseModel --> InsertLocalState["EcourtCaseController: cases.insert(0, saved)"]
    InsertLocalState --> SyncUpcomingHearings["Sort Cases & Update Upcoming Hearing Date Alerts"]
    SyncUpcomingHearings --> SuccessView(["Case Dashboard Refreshed with New Case Tracking"])

    class OpenScanner,ScanQR,ExtractCNR,APIRequestCNR,SelectLocation,SelectCourt,EnterCaseDetails,APIRequestManual,BackendScrape,ErrorState,ShowErrorToast,ReturnMultiList,DisplayPicker,UserSelectsCase,APIRequestSelect,FetchFullDetails,SaveToDB,ReturnCaseModel,InsertLocalState,SyncUpcomingHearings source;
    class ChooseMethod,ScrapeResult decision;
    class SuccessView success;
    class ShowErrorToast error;
```
