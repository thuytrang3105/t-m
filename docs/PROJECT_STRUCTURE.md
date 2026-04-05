# 🏗️ Kiến Trúc Tổng Thể Dự Án Store Lens

## 🤖 1. MODULE\_AI — Hệ Thống Phân Tích AI & Computer Vision

**Vai trò:** Đọc luồng video (RTSP từ camera), sử dụng các mô hình AI (YOLO, DeepSORT) để nhận diện người, theo dõi quỹ đạo, và phân tích các chỉ số (Heatmap, Dwell time). Kết quả được đẩy lên Redis.

```
MODULE_AI/
├── config/                     # AI Configurations (no source code)
│   ├── app_config.yaml         # Settings for FPS, resolution, AI thresholds (Confidence, NMS)
│   ├── zone_config.json        # ROI (Region of Interest) coordinate definitions
│   └── logging.yaml            # Log storage format and configuration
│
├── models/                     # Pre-trained model weights
│   ├── yolov8n_openvino/       # YOLOv8 (Intel Edge optimized .xml, .bin formats)
│   └── deepsort/               # Feature extraction Re-ID models (e.g., ckpt.t7).
│
├── src/                        # Main AI Source Code (4-Layer Architecture)
│   ├── core/                   # Layer 1: Perception (Detect & Track)
│   │   ├── detector.py         # Runs YOLOv8 on frames.
│   │   └── tracker.py          # Assigns unique IDs to objects (DeepSORT) 
│   │
│   ├── managers/               # Layer 2: State Management (Memory).
│   │
│   ├── analytics/              # Layer 3: Business Analytics Logic.
│   │
│   ├── communication/          # Layer 4: Data Transmission (Output).
│   │
│   └── utils/                  # Shared Utilities
│       ├── visualizer.py       # Debugging: Draws Bounding Boxes and Tracking IDs on video.
│       └── geometry.py         # Mathematical calculations (IoU, Euclidean distance).
│
├── docker/
│   └── Dockerfile              # AI Runtime Environment (OpenCV, OpenVINO, Python).
├── .env                        # Environment variables (RTSP Camera, Redis URL).
├── config.py                   # Global configuration loader class.
└── main.py                     # Entry Point: Camera reading loop and AI Pipeline execution
```

---

## ⚙️ 2. MODULE\_BE — Hệ Thống Backend API (Node.js/Express)

**Vai trò:** Là cầu nối trung tâm. Lắng nghe dữ liệu từ Redis (do AI đẩy lên), lưu trữ vào Database (MongoDB), và cung cấp các RESTful APIs cho Frontend hiển thị biểu đồ, quản lý Camera, User, v.v.

**Kiến trúc:** Layered Architecture — `Controller → Service → Model/Schema`

```
MODULE_BE/
├── src/
│   ├── api/
│   │   └── index.js                    # Main route export/import (optional).
│   │
│   ├── config/                         # External service connection setups
│   │   ├── databaseMonogo.js           # MongoDB connection config (Mongoose).
│   │   └── redis.js                    # Redis connection config (Subscribing to AI data).
│   │
│   ├── controllers/                    # HTTP Request/Response handling
│   │   └── auth.controller.js          # Logic for Login, Register, Refresh Token.
│   │
│   ├── middlewares/                    # Request filters
│   │   ├── auth.middleware.js          # JWT Authentication and Role-based access control.
│   │   └── morgan.middleware.js        # API request logging (Morgan).
│   │
│   ├── routes/                         # API Endpoint definitions (URLs)
│   │   ├── auth.routes.js              # Routes for /api/auth/....
│   │   └── index.route.js              # Aggregates all routes.
│   │
│   ├── schemas/                        # Database Models / Schema declarations (Mongoose).
│   │
│   ├── service/                        # Business Logic (decoupled from Controllers)
│   │   └── auth.service.js             # Token generation and password verification logic.
│   │
│   └── utils/                          # Shared utility functions
│       ├── catchAsync.js               # Automatic Try/Catch wrapper for Async/Await.
│       ├── exceptions.js               # Custom Error Classes management.
│       ├── handleToken.js              # JWT generation/verification logic.
│       ├── hashpassword.js             # Password hashing logic (Bcrypt).
│       └── response.js                 # Standardized API response format: { status, data, message }.
│
├── tests/
│   ├── setup.js                        # Test environment setup (Jest config).
│   └── units/                          # Unit testing for specific functions.
│
├── src/app.js                          # Express App initialization, Middlewares & Routes.
├── src/server.js                       # Entry point: Starts server listening on Port.
├── docker-compose.yml                  # Simultaneous deployment of BE, DB, and Redis.
└── package.json                        # Node.js library dependencies.
```

---

## 💻 3. MODULE\_FE — Hệ Thống Frontend (React.js/Vite)

**Vai trò:** Giao diện tương tác cho người dùng cuối (chủ cửa hàng, quản lý). Hiển thị Dashboard trực quan, bản đồ Heatmap, quản lý danh sách Camera, cho phép vẽ Zone (ROI) trực tiếp trên trình duyệt.

**Kiến trúc:** Feature-based Structure kết hợp Redux Toolkit.

```
MODULE_FE/
├── src/
│   ├── assets/
│   │   └── logo.png                        # Images, Icons, Fonts 
│   │
│   ├── components/                         # Reusable UI Components
│   │   ├── common/                         # Dumb components (primitive)
│   │   │   ├── Loading.jsx                 # Spinner 
│   │   │   ├── NotFound.jsx                # 404 Page 
│   │   │   ├── EmptyState.jsx              # "No Data" display 
│   │   │   └── ConfirmationModal.jsx       # Confirmation popups (Delete, Save) 
│   │   ├── Header.jsx                      # Top Header bar 
│   │   ├── Navbar.jsx                      # Navigation bar (Side-menu) 
│   │   └── Footer.jsx                      # Page Footer 
│   │
│   ├── features/                           # Smart components — Grouped by feature
│   │   ├── Authentication/
│   │   │   └── authen.jsx                  # Login / Registration 
│   │   ├── Dashboard/
│   │   │   └── Dashboard.jsx               # Overview screen (charts, summary metrics) 
│   │   ├── AnalyticsHeatmap/
│   │   │   └── AnalyticsHeatmapPage.jsx    # Heatmap viewing and filtering 
│   │   ├── AnalyticsDwellTime/
│   │   │   └── AnalyticsDwellTime.jsx      # Dwell time statistics 
│   │   ├── CameraZoneConfig/
│   │   │   └── CameraZoneConfig.jsx        # Camera stream view and Zone drawing (Polygon) 
│   │   ├── AssetManagement/
│   │   │   ├── ManagementProduct.jsx       # Camera/Device management (CRUD) 
│   │   │   ├── asset.slice.js              # Redux reducer for Asset state 
│   │   │   └── asset.thunk.js              # Redux thunk for asynchronous API calls 
│   │   └── UserManagement/
│   │       └── ManagerUsers.jsx            # Employee / Account management 
│   ├── layouts/
│   │   └── MainLayout.jsx                  # Main Layout (Header + Sidebar + Dynamic Content) 
│   │
│   ├── redux/
│   │   └── store.js                        # Redux Store initialization with Slices 
│   │
│   ├── routes/
│   │   └── index.jsx                       # URL routing (Public vs. Private routes) 
│   │
│   ├── styles/
│   │   └── index.css                       # Global Styles (TailwindCSS initialization) 
│   │
│   ├── App.jsx                             # App wrapper with Providers (Redux, Router, Theme) 
│   └── main.jsx                            # Entry point: Mounts React to the DOM 
│
├── index.html                              # Root HTML file 
├── eslint.config.js                        # Linting configuration 
├── vite.config.js                          # Vite configuration (Bundler) 
└── package.json                            # Frontend library list (React, Redux, Axios, Router) 
```
