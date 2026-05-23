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
        FeatureFlagService["FeatureFlagService & FeatureGate"]
    end

    subgraph BackendServices ["Backend & Realtime Services"]
        APIServer["Node.js API Server (REST)<br/>Base URL: https://api.lawl.app/api<br/>Port: 443 HTTPS"]
        SocketServer["Socket.IO Server (Signaling)<br/>Host: https://api.lawl.app<br/>Port: 443 WSS"]
        Database[("MongoDB Database")]
    end

    subgraph ThirdParty ["Third-Party Service Providers"]
        FCM["Firebase Cloud Messaging (FCM)"]
        HMS["100ms SDK (WebRTC Rooms)"]
        Razorpay["Razorpay Payment Gateway"]
        eCourtsScraper["eCourts Portal / Scraper"]
    end

    %% Client Relations
    ClientApp -- "HTTP REST APIs" --> APIServer
    ClientApp -- "Websocket WSS Connection" --> SocketServer
    LawyerApp -- "HTTP REST APIs" --> APIServer
    LawyerApp -- "Websocket WSS Connection" --> SocketServer
    FeatureFlagService -- "GET /config (Latency: 100-250ms)" --> APIServer

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
    class ClientApp,LawyerApp,PlatformChannel,NativeRingtone,FeatureFlagService client;
    class APIServer,SocketServer,Database server;
    class FCM,HMS,Razorpay,eCourtsScraper external;
```

---

## 2. Onboarding & Authentication Workflow
This flowchart outlines the process of user registration, role selection, and the lawyer verification pipeline (KYC approval/rejection) with exact API endpoints, request/response payload details, and average latencies.

```mermaid
flowchart TD
    %% Styling
    classDef step fill:#f8fafc,stroke:#64748b,stroke-width:1px,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#0f172a;
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d;
    classDef failure fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d;

    Start(["Start Onboarding / Login"]) --> InputPhone["Input Mobile Number"]
    InputPhone --> ReqOTP["Request OTP (API Call)"]
    
    subgraph OTP Generation API
        ReqOTP --> SendRegisterPhone["POST /auth/register/phone/send-otp<br/>Payload: { phone }<br/>Latency: Fast (~200-500ms)"]
        ReqOTP --> SendLoginPhone["POST /auth/login/phone/send-otp<br/>Payload: { phone }<br/>Latency: Fast (~200-500ms)"]
    end

    SendRegisterPhone --> SendSMS["SMS Gateway Dispatches OTP"]
    SendLoginPhone --> SendSMS
    
    SendSMS --> EnterOTP["User Enters OTP"]
    EnterOTP --> VerifyOTP{"OTP Valid?"}

    VerifyOTP -- No --> AlertError["Display Error Toast"]
    AlertError --> EnterOTP

    VerifyOTP -- Yes --> CheckProfile{"Profile Exists?<br/>POST /auth/login/phone/verify<br/>Payload: { phone, otp }<br/>Response: { success: true, token }<br/>Latency: Fast (~150-300ms)"}

    CheckProfile -- Yes (Mark Auth Complete) --> CheckRole{"Role Type?<br/>GET /user/profile<br/>Latency: Fast (~100-200ms)"}
    
    CheckProfile -- No (Create Temp Session) --> FillDetails["Fill Basic Details (Name, Email)<br/>POST /auth/register/phone/verify<br/>Payload: { phone, otp }<br/>Response: { success: true, token }<br/>Latency: Fast (~150-300ms)"]
    
    FillDetails --> ReqEmailOTP["Verify Email Address<br/>POST /auth/register/email/send-otp<br/>Payload: { email }<br/>Latency: Fast (~200-600ms)<br/>POST /auth/register/email/verify<br/>Payload: { email, otp }<br/>Response: { success: true, token }<br/>Latency: Fast (~150-300ms)"]
    
    ReqEmailOTP --> ChooseRole{"Select Role"}
    
    ChooseRole -- "Client User" --> SaveClient["Create Client Profile<br/>POST /user/register<br/>Payload: { name }<br/>Response: { success: true, token }<br/>Latency: Fast (~150-300ms)"]
    SaveClient --> ClientDashboard(["Access Client Dashboard"])
    
    ChooseRole -- "Lawyer / Advocate" --> SaveLawyer["Create Basic Lawyer Profile<br/>POST /lawyer/register<br/>Payload: { name, barCouncilNumber, state, district, courtComplex }<br/>Response: { success: true, token }<br/>Latency: Fast (~200-400ms)"]
    
    CheckRole -- "Client User" --> ClientDashboard
    CheckRole -- "Lawyer / Advocate" --> CheckVerificationStatus
    
    SaveLawyer --> UploadKYC["Upload KYC Docs<br/>POST /lawyer/upload-document (Multipart)<br/>Payload: [document: file, type: 'registration']<br/>POST /lawyer/ekyc<br/>Payload: { specialization, experienceYears, languages }<br/>Latency: Medium (~300-1200ms)"]
    UploadKYC --> CheckVerificationStatus{"Verification Status?<br/>GET /lawyer/profile<br/>Latency: Fast (~100-200ms)"}
    
    CheckVerificationStatus -- "Approved" --> LawyerDashboard(["Access Lawyer Dashboard"])
    CheckVerificationStatus -- "Pending" --> BlockScreen["Show Pending Verification Overlay"]
    CheckVerificationStatus -- "Rejected" --> ReUploadKYC["Show Rejection Reason & Re-upload Request"]
    
    BlockScreen --> ExitOrWait["Wait for Admin Approval"]
    ReUploadKYC --> UploadKYC

    class InputPhone,ReqOTP,SendRegisterPhone,SendLoginPhone,SendSMS,EnterOTP,AlertError,FillDetails,ReqEmailOTP,SaveClient,SaveLawyer,UploadKYC,BlockScreen,ReUploadKYC,ExitOrWait step;
    class VerifyOTP,CheckProfile,CheckRole,ChooseRole,CheckVerificationStatus decision;
    class ClientDashboard,LawyerDashboard success;
```

---

## 3. Session Lifecycle & Socket Signaling
Consultations in LAWL are structured around a **Session-First** flow, where chat/call features are only enabled inside an active session. State transitions are requested via secure HTTP endpoints, and real-time updates are pushed via Socket.IO.

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client User
    participant API as Node.js API Server
    participant DB as MongoDB Database
    participant Socket as Socket.IO Server
    actor Lawyer as Lawyer User

    %% Request Initiation
    Client->>API: HTTP POST /sessions<br/>Payload: { lawyerId, chatId, ratePerMinute, type }<br/>Latency: Fast (~150-300ms)
    activate API
    API->>DB: Create Session (status: "REQUESTED")
    API->>Socket: Trigger real-time alert
    Socket-->>Lawyer: Socket emit("new_session_request", sessionPayload)
    API-->>Client: Response { success: true, session }
    deactivate API
    Note over Lawyer: Lawyer receives request list in real-time

    %% Accept or Decline decision
    alt Lawyer accepts session request
        Lawyer->>API: HTTP POST /sessions/:sessionId/accept<br/>Latency: Fast (~200-350ms)
        activate API
        API->>DB: Update Session (status: "ACTIVE")
        API->>Socket: Trigger active session state
        Socket-->>Client: Socket emit("session_active", sessionPayload)
        Socket-->>Lawyer: Socket emit("session_active", sessionPayload)
        API-->>Lawyer: Response { success: true }
        deactivate API
        Note over Client, Lawyer: Chat screen unlocked, Call features active
    else Lawyer rejects session request
        Lawyer->>API: HTTP POST /sessions/:sessionId/reject<br/>Latency: Fast (~150-300ms)
        activate API
        API->>DB: Update Session (status: "REJECTED")
        API->>Socket: Trigger reject alert
        Socket-->>Client: Socket emit("session_rejected", { sessionId })
        API-->>Lawyer: Response { success: true }
        deactivate API
        Note over Client: Notified of rejection, refund credited
    else Client cancels request before response
        Client->>API: HTTP POST /sessions/:sessionId/cancel<br/>Latency: Fast (~150-300ms)
        activate API
        API->>DB: Update Session (status: "CANCELLED")
        API->>Socket: Trigger cancel alert
        Socket-->>Lawyer: Socket emit("session_cancelled", { sessionId })
        API-->>Client: Response { success: true }
        deactivate API
    end

    %% Session Ending
    Note over Client, Lawyer: Active consultation in progress...
    alt User/Lawyer ends session manually
        Client->>API: HTTP POST /sessions/:sessionId/end<br/>Latency: Fast (~200-400ms)
        activate API
        API->>DB: Update Session (status: "ENDED")
        API->>Socket: Trigger end session
        Socket-->>Lawyer: Socket emit("session_ended", { sessionId, reason: "USER_ENDED" })
        Socket-->>Client: Socket emit("session_ended", { sessionId, reason: "USER_ENDED" })
        API-->>Client: Response { success: true }
        deactivate API
        Note over Client, Lawyer: Navigate to feedback / ratings screen (POST /sessions/:sessionId/rate)
    end
```

---

## 4. Call Lifecycle & Native Ringtone Flow
This sequence diagram details how calls are initiated, how the native system ringtone loops are activated in foreground/background states, how WebRTC is joined using the 100ms SDK, and how teardown works.

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client User
    participant Socket as Socket.IO Server
    participant API as Node.js API Server
    participant FCM as Firebase Push Server
    participant LawyerCtrl as Lawyer CallController (GetX)
    participant LocalNotif as LocalNotificationService
    participant Native as Native Android (MainActivity)
    participant HMS as 100ms SDK

    Client->>Socket: Socket emit("request_call", { sessionId, callType })
    
    %% Foreground vs Background Branching
    Note over Socket, LawyerCtrl: Scenario A: Lawyer App is in Foreground
    Socket-->>LawyerCtrl: Socket emit("call_incoming", callPayload)
    
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
        
        LawyerCtrl->>API: HTTP POST /sessions/:sessionId/accept<br/>Latency: Fast (~200-350ms)
        API-->>LawyerCtrl: Response { success: true }
        API->>Socket: Broadcast session_active
        Socket-->>Client: Socket emit("session_active", callPayload)
        
        %% WebRTC Join Room
        par Client Joins Room
            Client->>API: HTTP GET /sessions/:sessionId/token<br/>Latency: Medium (~300-600ms)
            API-->>Client: Response { success: true, token }
            Client->>HMS: Join room (token)
        and Lawyer Joins Room
            LawyerCtrl->>API: HTTP GET /sessions/:sessionId/token<br/>Latency: Medium (~300-600ms)
            API-->>LawyerCtrl: Response { success: true, token }
            LawyerCtrl->>HMS: Join room (token)
        end
        
        Note over Client, HMS: WebRTC Call established (Video/Voice Active)

    else Lawyer Declines Call
        LawyerCtrl->>Native: MethodChannel("stopRingtone")
        activate Native
        Native->>Native: ringtone.stop()
        deactivate Native
        LawyerCtrl->>API: HTTP POST /sessions/:sessionId/reject<br/>Latency: Fast (~150-300ms)
        API-->>LawyerCtrl: Response { success: true }
        API->>Socket: Broadcast session_rejected
        Socket-->>Client: Socket emit("session_rejected", { sessionId })
        LawyerCtrl->>LawyerCtrl: Navigate back / Close call screen

    else Call Timeout (No Answer for 30s)
        Socket-->>Client: Socket emit("call_timeout", { sessionId })
        Socket-->>LawyerCtrl: Socket emit("call_cancelled", { sessionId })
        LawyerCtrl->>Native: MethodChannel("stopRingtone")
        activate Native
        Native->>Native: ringtone.stop()
        deactivate Native
        LawyerCtrl->>LawyerCtrl: Close call screen & show missed call
    end

    %% Call Ending / Hanging Up
    Note over Client, HMS: Active call conversation in progress...
    alt Client or Lawyer Hangs Up Call
        Client->>Socket: Socket emit("end_call", { sessionId, reason: "USER_ENDED" })
        Socket-->>LawyerCtrl: Socket emit("call_ended", { sessionId, reason: "USER_ENDED" })
        par Client Teardown
            Client->>HMS: Leave room / End room
            Client->>Native: Clear window flags & reset state
        and Lawyer Teardown
            LawyerCtrl->>HMS: Leave room / End room
            LawyerCtrl->>Native: MethodChannel("stopRingtone") & clear flags
        end
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
    
    TimerCheck -- Yes --> EmitWarning["Socket Server Emits 'session_warning'<br/>Payload: { sessionId, remainingSeconds, blockPrice, canExtend, userWalletBalance }"]
    
    EmitWarning --> ShowBottomSheet["Flutter UI: Display SessionExtensionModal"]
    
    ShowBottomSheet --> CheckBalance{"Client Wallet Balance >= Block Price?"}
    
    CheckBalance -- Yes --> ConfirmExtend["Client Taps 'Confirm Extension (+15m)'"]
    ConfirmExtend --> EmitExtend["HTTP POST /sessions/:sessionId/extend<br/>Latency: Fast (~250-500ms)"]
    EmitExtend --> DBUpdate["Server Deducts Balance & Adds 900s in DB"]
    DBUpdate --> SuccessAck["Socket Server Emits 'extension_success' to both parties"]
    SuccessAck --> ExtendCountdown["Flutter Updates Timer UI (+15 Mins)"]
    ExtendCountdown --> ActiveSession

    CheckBalance -- No --> PromptTopUp["Flutter UI Shows Top-Up Required State (Shortfall)"]
    
    PromptTopUp --> SelectAmount["User Chooses Top-Up Amount (Shortfall or default)"]
    SelectAmount --> InitRazorpay["POST /payments/create-order<br/>Payload: { amount }<br/>Response: { success, order_id, key_id }<br/>Latency: Medium (~300-600ms)"]
    
    InitRazorpay --> RazorpayOverlay["Launch Razorpay checkout overlay SDK"]
    RazorpayOverlay --> RazorpayProcess{"Razorpay Payment Success?"}
    
    RazorpayProcess -- No --> PaymentFailed["Display Payment Error Toast"]
    PaymentFailed --> ShowBottomSheet
    
    RazorpayProcess -- Yes --> RazorpayCallback["Razorpay SDK Returns Payment ID, Order ID & Signature"]
    RazorpayCallback --> VerifyPayment["POST /payments/verify<br/>Payload: { razorpay_order_id, razorpay_payment_id, razorpay_signature }<br/>Latency: Medium (~350-700ms)"]
    VerifyPayment --> APIDBUpdate["Server Validates Signature, Credits Wallet, Returns Success"]
    APIDBUpdate --> APISuccessAck["API returns success and triggers POST /sessions/:sessionId/extend automatically"]
    APISuccessAck --> ExtendCountdown

    TimerCheck -- Expiry reached (0s) --> EndSession["Socket Server Emits 'session_ended'<br/>Payload: { sessionId, reason: 'OUT_OF_BALANCE' }"]
    EndSession --> TerminateTracks["100ms Audio/Video Tracks Terminated"]
    TerminateTracks --> GoToFeedback(["Close Session & Navigate to Ratings Page"])

    class ActiveSession,EmitWarning,ShowBottomSheet,ConfirmExtend,EmitExtend,DBUpdate,SuccessAck,ExtendCountdown,PromptTopUp,SelectAmount,InitRazorpay,RazorpayOverlay,RazorpayCallback,VerifyPayment,APIDBUpdate,APISuccessAck,PaymentFailed,EndSession,TerminateTracks state;
    class TimerCheck,CheckBalance,RazorpayProcess decision;
    class GoToFeedback action;
```

---

## 6. eCourts Scraper & Case Management Flow
This diagram details how advocates can search and add cases to their dashboard by either scanning a physical QR code or performing a query against state/district filters, scraper logic, and multi-result picking. Captchas and session cookies are automated backend-side.

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
    ExtractCNR --> APIRequestCNR["HTTP POST /cases/ecourt/scan<br/>Payload: { cnr }<br/>Latency: Slow/Scraping (~3.0s - 12.0s)"]
    
    ChooseMethod -- "Manual Filter Search" --> SelectLocation["Select State & District"]
    SelectLocation --> SelectCourt["Select Court Complex"]
    SelectCourt --> EnterCaseDetails["Enter Case Type, Case Number, & Year"]
    EnterCaseDetails --> APIRequestManual["HTTP POST /cases/ecourt/search<br/>Payload: { state_code, dist_code, court_complex_code, case_type, case_no, year }<br/>Latency: Slow/Scraping (~3.0s - 15.0s)"]

    APIRequestCNR --> BackendScrape["Backend Scraper Solves Captchas & Scrapes eCourts Portal"]
    APIRequestManual --> BackendScrape
    
    BackendScrape --> ScrapeResult{"Parse Scrape Results"}
    
    ScrapeResult -- "Error / Not Found" --> ErrorState["Return Scrape Failed Event"]
    ErrorState --> ShowErrorToast["Flutter UI: Display Error Alert Banner"]
    ShowErrorToast --> StartCaseAdd

    ScrapeResult -- "Multiple Cases Match" --> ReturnMultiList["Return Matching Cases Index List<br/>Response: { success: true, multiple: true, cases: [...] }"]
    ReturnMultiList --> DisplayPicker["Flutter UI: Display Selection Picker Sheet"]
    DisplayPicker --> UserSelectsCase["Lawyer Selects Exact Target Case"]
    UserSelectsCase --> APIRequestSelect["HTTP POST /cases/ecourt/select<br/>Payload: { caseInfo }<br/>Latency: Slow/Scraping (~2.0s - 8.0s)"]
    APIRequestSelect --> FetchFullDetails["Backend Scrapes Selected Case Details Page"]
    FetchFullDetails --> SaveToDB

    ScrapeResult -- "Single Direct Match" --> SaveToDB["Backend Saves Case Model to MongoDB Database"]
    
    SaveToDB --> ReturnCaseModel["Return Structured CaseModel JSON<br/>Response: { success: true, case }"]
    ReturnCaseModel --> InsertLocalState["EcourtCaseController: cases.insert(0, saved)"]
    InsertLocalState --> SyncUpcomingHearings["Sort Cases & Update Upcoming Hearing Date Alerts"]
    SyncUpcomingHearings --> SuccessView(["Case Dashboard Refreshed with New Case Tracking"])

    class OpenScanner,ScanQR,ExtractCNR,APIRequestCNR,SelectLocation,SelectCourt,EnterCaseDetails,APIRequestManual,BackendScrape,ErrorState,ShowErrorToast,ReturnMultiList,DisplayPicker,UserSelectsCase,APIRequestSelect,FetchFullDetails,SaveToDB,ReturnCaseModel,InsertLocalState,SyncUpcomingHearings source;
    class ChooseMethod,ScrapeResult decision;
    class SuccessView success;
    class ShowErrorToast error;
```

---

## 7. Feature Gates & Configuration System
To support flexible toggles and region-specific launches, LAWL implements a dynamic Feature Gate system loading configurations from the backend.

### 7.1 Architecture Overview
- **Startup Fetching**: The `SplashScreen` fetches the master flags during initialization via the REST API endpoint `GET /config` (Latency: ~100-250ms).
- **Caching Mechanism**: The `FeatureFlagService` persists the key-value toggles locally using secure storage (`FlutterSecureStorage` under cache key `feature_flags_cache`). This prevents visual flickering when navigating screens.
- **Auto-Refresh**: Toggles are automatically re-fetched in the background on application resume events via `WidgetsBindingObserver`.
- **Runtime Evaluation**: UI widgets evaluate flags using the static class `FeatureGate.isEnabled(key)` or `FeatureGate.evaluate(key)`. The evaluation yields specific `FeatureBehavior` parameters (e.g. `show`, `hide`, `showComingSoon`, `showDisabledState`).

### 7.2 Feature Flags & Code References

#### 1. `location_map_enabled` (Boolean)
- **Description**: Governs the interactive client-side map displaying available nearby advocates.
- **FAB Toggle (`user_main_screen.dart:L85`)**: If evaluated to `false`, the bottom center Floating Action Button is completely hidden (returns `SizedBox.shrink()`).
- **Screen Gate (`lawyers_near_me_screen.dart:L179`)**: If user navigates directly and the flag is disabled, it bypasses permission requests and renders a fallback "Map View Unavailable" warning screen. If enabled, it initializes geolocation trackers and requests nearby lawyers from the endpoint:
  ```http
  GET /lawyer/nearby?lat={latitude}&lng={longitude}&radiusKm=20
  Headers: Authorization: Bearer {token}
  Latency: Fast (~150-300ms)
  ```

#### 2. `google_calendar` (Boolean)
- **Description**: Dictates whether lawyer calendar views show external Google Calendar sync utilities.
- **Calendar Integration (`lawyer_calendar_screen.dart:L419`)**: When set to `true`, the collapsing persistent calendar sliver displays the styled outline button to connect and synchronize calendar integrations:
  ```dart
  // Triggers Google OAuth callback integration:
  _ctrl.connectGoogleCalendar()
  ```
  If set to `false`, the synchronization button is skipped from rendering.

### 7.3 Lifecycle of Feature Flags

```mermaid
graph TD
    Splash["SplashScreen Init"] --> FetchFlags["API Call: GET /config<br/>(Latency: 100-250ms)"]
    FetchFlags --> ResponseCheck{"API Response Success?"}
    
    ResponseCheck -- Yes --> UpdateInMemory["Update FeatureFlagService.flags Map"]
    UpdateInMemory --> WriteCache["Save JSON string in FlutterSecureStorage<br/>(feature_flags_cache)"]
    
    ResponseCheck -- No --> ReadCache["Read cached JSON from FlutterSecureStorage"]
    ReadCache --> RestoreInMemory["Restore FeatureFlagService.flags Map"]
    
    WriteCache --> AppReady["Unblock App Startup & Render Main Screen"]
    RestoreInMemory --> AppReady
    
    %% Evaluation
    AppReady --> UIBuild["UI Build / Widget Build"]
    UIBuild --> EvaluateFlag{"FeatureGate.isEnabled(key)"}
    
    EvaluateFlag -- "location_map_enabled == true" --> RenderFAB["Render Golden Floating Location FAB"]
    EvaluateFlag -- "location_map_enabled == false" --> HideFAB["Render Empty SizedBox.shrink()"]
    
    EvaluateFlag -- "google_calendar == true" --> RenderGCalBtn["Render 'Connect Google Calendar' Button"]
    EvaluateFlag -- "google_calendar == false" --> HideGCalBtn["Do Not Render Button"]
```
