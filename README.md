# Flowboard

> Không gian làm việc đồ thị node (Infinite-Canvas Workspace) cục bộ cho quy trình tạo media bằng AI (Google Flow, Veo 3.1, Imagen, LLM).

---

## 1. Chức năng chính (Features)

- **Canvas đồ thị node tương tác (Node-based Graph Canvas)**:
  - Xây dựng quy trình sáng tạo đa phương tiện theo dạng đồ thị có hướng (DAG).
  - Tái sử dụng node, quản lý quan hệ phụ thuộc dữ liệu và tái tạo độc lập từng biến thể (variants).
- **Quản lý Reference & Nhân vật (Character & Visual Asset)**:
  - **Character**: Cố định khuôn mặt, giới tính, quốc tịch để giữ tính nhất quán nhận diện nhân vật qua các video i2v.
  - **Visual Asset**: Tải lên sản phẩm/vật thể hoặc tạo từ prompt, hỗ trợ tinh chỉnh (Refine/edit_image) giữ nguyên sản phẩm gốc.
- **Ghép bố cục đa nguồn (Composition & Multi-Ref Image Generation)**:
  - Kết nối nhiều Reference (Character + Product + Background) vào một node Image.
  - Sinh hàng loạt 1–4 biến thể với các góc dáng (pose) khác nhau trong cùng một lần chạy.
- **Biến ảnh thành video (Veo 3.1 Image-to-Video)**:
  - Chuyển đổi ảnh thành clip chuyển động 8 giây qua Google Veo 3.1 i2v.
  - **Multi-source batch**: Tự động tạo 4 video tương ứng từ 4 biến thể ảnh chỉ với 1 click.
  - Chế độ camera cố định (Static - chuẩn e-commerce) hoặc chuyển động linh hoạt (Dynamic).
- **Tự động sinh Prompt & Phân tích hình ảnh (Auto-Prompt & Vision)**:
  - Phân tích ngữ cảnh hình ảnh upstream bằng Vision (Multimodal LLM) để lưu thành `aiBrief`.
  - Tự động tạo prompt thời trang/điện ảnh theo phong cách editorial ăn khớp với bối cảnh (Studio, đường phố, quán café, ngoại cảnh).
- **Kịch bản phân cảnh (Storyboard Node)**:
  - Xây dựng chuỗi 1–8 khung hình liên tục theo kịch bản (roots và continuations song song).
- **Hỗ trợ đa mô hình AI (Multi-LLM Provider Layer)**:
  - Tích hợp linh hoạt với các CLI: **Claude Code** (`claude`), **Gemini** (Antigravity CLI `agy`), hoặc **OpenAI Codex** (`codex`).
  - Cấu hình độc lập model và reasoning effort cho từng tác vụ: Auto-Prompt, Vision, Planner.
- **Activity Feed thời gian thực**:
  - Theo dõi nhật ký tiến trình từng tác vụ backend (gen image, gen video, prompt synth, vision).
  - Xem chi tiết request payload, kết quả, log lỗi và hủy tác vụ đang chạy.

---

## 2. Kiến trúc hệ thống (System Architecture)

```
┌─────────────────────────────────┐           ┌───────────────────────────────────┐
│     Chrome Extension (MV3)      │           │        React Frontend (Vite)      │
│  - Chạy trên flow.google.com    │           │  - ReactFlow 12 Canvas            │
│  - batchexecute RPC ký tự động  │           │  - Zustand Store                  │
│  - Sinh token reCAPTCHA hợp lệ  │           │  - Cổng: http://127.0.0.1:5173     │
└───────────────▲─────────────────┘           └─────────────────▲─────────────────┘
                │                                               │
           WebSocket                                          HTTP
            (:8355)                                          (:8434)
                │                                               │
                ▼                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          FastAPI Agent (Python Backend)                         │
│                          Địa chỉ: http://127.0.0.1:8434                         │
│  - REST API Routes (boards, nodes, edges, prompt, vision, activity, llm)        │
│  - Worker Queue xử lý tác vụ ngầm (in-process queue)                            │
│  - WebSocket Server (:8355) điều phối Chrome Extension                          │
│  - LLM Subprocess Bridge: gọi trực tiếp CLI (claude / agy / codex)              │
└───────────────────────┬─────────────────────────────────────────┬───────────────┘
                        │                                         │
                        ▼                                         ▼
         ┌──────────────────────────────┐          ┌──────────────────────────────┐
         │     SQLite Database          │          │      Local Storage / Media   │
         │  - storage/flowboard.db      │          │  - storage/media/            │
         │  - Lưu Board, Node, Edge,    │          │  - Cache ảnh, video tải từ   │
         │    Request, Asset, Plan      │          │    CDN của Google Flow       │
         └──────────────────────────────┘          └──────────────────────────────┘
```

### Nguyên lý hoạt động các thành phần:
1. **Frontend**: Giao diện đồ thị node vô hạn xây dựng trên ReactFlow và Zustand, giao tiếp hoàn toàn với Agent qua HTTP REST API.
2. **Agent (Backend)**: Đóng vai trò bộ não trung tâm quản lý cơ sở dữ liệu SQLite, hàng đợi tác vụ, gọi LLM CLI nội bộ và điều phối lệnh tạo media sang extension.
3. **Chrome Extension (MV3 Bridge)**: Vì Google Flow sử dụng cơ chế bảo mật nghiêm ngặt (`flow.google.com` dùng session cookie nội bộ, token per-page và reCAPTCHA một lần), Extension được nạp trực tiếp vào tab Flow đang mở để ký và thực thi lệnh `batchexecute` thay mặt cho Agent.
4. **Storage Cục bộ**: Toàn bộ dữ liệu nằm trên máy người dùng, bao gồm database SQLite và thư mục media cache.

---

## 3. Cấu trúc dự án (Project Structure)

```
flowboard/
├── agent/                      # Backend FastAPI (Python 3.10+)
│   ├── flowboard/
│   │   ├── db/                 # Định nghĩa schema CSDL (SQLModel: Board, Node, Edge, Request, ...)
│   │   ├── routes/             # Các API Endpoint RESTful
│   │   ├── services/           # Nghiệp vụ: Flow SDK, LLM bridge, prompt synth, vision describe
│   │   │   ├── flow_batch.py   # Bộ mã hóa/giải mã batchexecute RPC của Flow
│   │   │   ├── flow_client.py  # WebSocket client giao tiếp với Chrome Extension
│   │   │   ├── flow_sdk.py     # Tầng nghiệp vụ tương tác Google Flow
│   │   │   └── llm/            # Tầng provider đa mô hình (Claude, Gemini/agy, OpenAI Codex)
│   │   ├── worker/             # Hàng đợi tác vụ tạo ảnh, video, upload
│   │   ├── config.py           # Quản lý cấu hình biến môi trường
│   │   └── main.py             # Điểm khởi tạo FastAPI app & WebSocket listener
│   ├── tests/                  # Bộ kiểm thử tự động pytest
│   ├── pyproject.toml          # Khai báo gói và dependencies của agent
│   └── requirements.txt        # Danh sách thư viện Python
│
├── frontend/                   # Giao diện người dùng Web (React + Vite + TypeScript)
│   ├── src/
│   │   ├── canvas/             # Canvas đồ thị (Board.tsx, NodeCard.tsx, AddNodePalette.tsx, ...)
│   │   ├── components/         # Các modal cấu hình, activity bell, toolbar, dialogs
│   │   ├── store/              # Quản lý state toàn cục với Zustand (board, generation, settings)
│   │   └── api/                # HTTP client gọi tới backend Agent
│   ├── package.json            # Cấu hình dự án Node.js và dependencies
│   └── vite.config.ts          # Cấu hình Vite & proxy tới Agent backend
│
├── extension/                  # Chrome Extension Manifest V3 (Cầu nối với Google Flow)
│   ├── manifest.json           # Khai báo quyền, content scripts và background service worker
│   ├── background.js           # Kết nối WebSocket tới Agent (:8355), chuyển tiếp yêu cầu
│   └── content.js              # Inject vào flow.google.com để ký RPC và lấy reCAPTCHA
│
├── docs/                       # Tài liệu hướng dẫn, migration và assets
├── storage/                    # Thư mục lưu database SQLite và tệp media tải về (tự tạo khi chạy)
├── .env.example                # Tệp mẫu cấu hình biến môi trường
└── Makefile                    # Tập lệnh tiện ích quản lý và chạy dự án (Linux/macOS)
```

---

## 4. Các đường dẫn & Cổng dịch vụ (Routes & Ports)

### Cổng dịch vụ (Ports & Endpoints)
| Thành phần | Địa chỉ mặc định | Ghi chú |
|---|---|---|
| **Frontend Web** | `http://127.0.0.1:5173` | Giao diện React canvas chạy bằng Vite |
| **Agent Backend API** | `http://127.0.0.1:8434` | REST API FastAPI |
| **Extension WebSocket** | `ws://127.0.0.1:8355` | Cổng kết nối giữa Agent và Chrome Extension |
| **Google Flow** | `https://flow.google.com` | Nền tảng Flow cần mở sẵn và đăng nhập trong Chrome |

### Danh sách API Routes chính (Agent Backend)
| Phương thức | Đường dẫn API | Chức năng |
|---|---|---|
| `GET` | `/api/health` | Kiểm tra tình trạng backend và kết nối WebSocket với Extension |
| `GET` | `/api/auth/me` | Lấy thông tin tài khoản và gói cước Google Flow (`FLOWBOARD_PAYGATE_TIER`) |
| `GET`, `POST` | `/api/boards` | Danh sách và tạo mới bảng vẽ (Board) |
| `GET`, `PATCH`, `DELETE`| `/api/boards/{id}` | Lấy chi tiết, đổi tên hoặc xóa bảng vẽ |
| `GET`, `POST` | `/api/boards/{id}/nodes` | Lấy danh sách hoặc tạo node mới trên canvas |
| `PATCH`, `DELETE`| `/api/boards/{id}/nodes/{node_id}` | Cập nhật dữ liệu hoặc xóa node |
| `GET`, `POST` | `/api/boards/{id}/edges` | Quản lý các đường nối dữ liệu giữa các node |
| `DELETE` | `/api/boards/{id}/edges/{edge_id}` | Xóa đường nối giữa 2 node |
| `GET`, `POST` | `/api/boards/{id}/requests` | Danh sách tác vụ đang chạy hoặc kích hoạt tạo ảnh/video |
| `POST` | `/api/prompt/auto` | Tự động sinh prompt từ các node đầu vào |
| `POST` | `/api/prompt/auto-batch` | Tự động sinh danh sách N prompt cho các biến thể khác nhau |
| `POST` | `/api/vision/describe` | Phân tích thị giác hình ảnh và tạo tóm tắt mô tả |
| `GET`, `POST` | `/api/llm/providers` | Quản lý, kiểm tra kết nối và cấu hình mô hình LLM |
| `GET` | `/api/activity` | Lấy danh sách nhật ký hoạt động gần nhất |
| `POST` | `/api/upload` | Tải lên tệp ảnh/tài nguyên cục bộ |
| `GET` | `/api/media/{filename}` | Truy xuất hình ảnh/video đã được lưu trữ cục bộ |

---

## 5. Hướng dẫn chạy dự án (How to Run)

### Yêu cầu tiên quyết
1. **Python 3.10+**: Để chạy Agent backend.
2. **Node.js 20+**: Để chạy Frontend Vite.
3. **Google Chrome** và tài khoản Google Flow đã đăng ký gói **Pro** hoặc **Ultra** (Gói miễn phí không hỗ trợ Veo i2v / GEM_PIX_2).
4. **Một CLI LLM cài sẵn trên PATH**: Claude Code (`claude`), Antigravity CLI (`agy`), hoặc OpenAI Codex (`codex`).

---

### Bước 1: Cấu hình biến môi trường (.env)
Sao chép tệp mẫu `.env.example` thành `.env` tại thư mục gốc của dự án:
```powershell
# Trên Windows PowerShell:
Copy-Item .env.example .env

# Trên Linux/macOS:
cp .env.example .env
```
Mở `.env` và điền 2 giá trị bắt buộc:
```ini
# ID của một Project tạo sẵn trên https://flow.google.com (lấy từ URL trên trình duyệt)
FLOWBOARD_FLOW_PROJECT_ID=dien-id-project-tai-day

# Gói cước Flow: PAYGATE_TIER_TWO (Ultra) hoặc PAYGATE_TIER_ONE (Pro)
FLOWBOARD_PAYGATE_TIER=PAYGATE_TIER_TWO
```

---

### Bước 2: Cài đặt Chrome Extension
1. Mở trình duyệt Chrome và truy cập: `chrome://extensions/`.
2. Bật chế độ **Developer mode** (Góc trên cùng bên phải).
3. Nhấp nút **Load unpacked** (Tải tiện ích đã giải nén) và chọn thư mục `extension/` trong dự án.
4. Mở một tab mới tới [https://flow.google.com/](https://flow.google.com/), đăng nhập tài khoản và **giữ nguyên tab này luôn mở**.

---

### Bước 3: Khởi chạy Agent Backend (FastAPI)
Mở một cửa sổ Terminal mới:

#### Trên Windows (PowerShell):
```powershell
cd agent
python -m pip install -r requirements.txt
python -m pip install -e .
python -m uvicorn flowboard.main:app --reload --port 8434
```

#### Trên Linux / macOS:
```bash
cd agent
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install -e .
uvicorn flowboard.main:app --reload --port 8434
```
*(Kiểm tra kiểm tra hoạt động tại: `http://127.0.0.1:8434/api/health`)*

---

### Bước 4: Khởi chạy Frontend (React + Vite)
Mở một cửa sổ Terminal khác:
```bash
cd frontend
npm install
npm run dev
```

---

### Bước 5: Sử dụng Flowboard
1. Truy cập giao diện tại: **`http://localhost:5173`**.
2. Kiểm tra biểu tượng Extension trên thanh công cụ chuyển sang màu xanh lá (`●`), báo hiệu đã kết nối thành công với Agent.
3. Kéo thả các node **Character**, **Visual asset**, **Image**, **Video**, kết nối đường dây và bấm **▶ Generate** để bắt đầu quy trình sáng tạo.
