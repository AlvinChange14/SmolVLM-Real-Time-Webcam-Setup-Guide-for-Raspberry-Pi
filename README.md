# SmolVLM Real-Time Webcam Setup Guide for Raspberry Pi

This guide provides step-by-step instructions to set up and run the SmolVLM real-time webcam project on a Raspberry Pi. The project uses a webcam to capture images and processes them with the SmolVLM-500M-Instruct model to generate descriptions.

## Prerequisites
- Raspberry Pi with Raspberry Pi OS (Linux-based) installed.
- A USB webcam connected to the Raspberry Pi.
- Internet access for downloading packages and model files.

## Step-by-Step Instructions

### 1. Update the System
Update the package lists and upgrade all installed packages to their latest versions.

```bash
sudo apt update
sudo apt full-upgrade -y
```

### 2. Install Python and Git
Install Python 3, pip, and Git to manage the project dependencies and clone repositories.

```bash
sudo apt install -y python3 python3-pip git
```

Verify the Python version:

```bash
python3 --version
```

### 3. Install Webcam Dependencies
Install `fswebcam` and `v4l-utils` to interface with the webcam.

```bash
sudo apt install -y fswebcam v4l-utils
```

### 4. Verify Webcam Connection
Check if the webcam is detected:

```bash
lsusb
```

List available video devices:

```bash
ls /dev/video*
```

Test the webcam by capturing a sample image:

```bash
fswebcam -d /dev/video0 -r 640x480 test.jpg
```

Check supported webcam formats:

```bash
v4l2-ctl --list-formats-ext
```

### 5. Create Project Directory
Create a directory for the project:

```bash
mkdir -p ~/smolvlm_project
cd ~/smolvlm_project
```

### 6. Clone the SmolVLM Webcam Repository
Clone the real-time webcam project repository:

```bash
git clone https://github.com/ngxson/smolvlm-realtime-webcam.git
cd smolvlm-realtime-webcam
```

### 7. Install Python Dependencies
Install required Python libraries for image processing and HTTP requests:

```bash
sudo apt install -y python3-opencv python3-requests python3-pil
```

Verify the installed library versions:

```bash
python3 -c "import cv2; print(cv2.__version__)"
python3 -c "import requests; print(requests.__version__)"
python3 -c "from PIL import Image; print(Image.__version__)"
```

### 8. Install Build Tools
Install tools required to build llama.cpp:

```bash
sudo apt install -y build-essential cmake
```

### 9. Clone and Build llama.cpp
Clone the llama.cpp repository and build it:

```bash
cd ~/smolvlm_project
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
```

Install additional dependencies:

```bash
sudo apt update
sudo apt install -y libcurl4-openssl-dev
```

Build llama.cpp:

```bash
rm -rf build
mkdir build
cd build
cmake ..
cmake --build . --config Release
```

Verify the build:

```bash
./bin/llama-cli --version
```

### 10. Download SmolVLM Model Files
Download the SmolVLM-500M-Instruct model and its multimodal projection file:

```bash
wget https://huggingface.co/ggml-org/SmolVLM-500M-Instruct-GGUF/resolve/main/SmolVLM-500M-Instruct-Q8_0.gguf -O ~/smolvlm_project/SmolVLM-500M-Instruct-Q8_0.gguf
wget https://huggingface.co/ggml-org/SmolVLM-500M-Instruct-GGUF/resolve/main/mmproj-SmolVLM-500M-Instruct-Q8_0.gguf -O ~/smolvlm_project/mmproj-SmolVLM-500M-Instruct-Q8_0.gguf
```

Verify the downloaded files:

```bash
ls -lh ~/smolvlm_project/SmolVLM-500M-Instruct-Q8_0.gguf
ls -lh ~/smolvlm_project/mmproj-SmolVLM-500M-Instruct-Q8_0.gguf
```

### 11. Start the llama.cpp Server
Run the llama.cpp server with the SmolVLM model:

```bash
cd ~/smolvlm_project/llama.cpp/build/bin
./llama-server -m ~/smolvlm_project/SmolVLM-500M-Instruct-Q8_0.gguf --mmproj ~/smolvlm_project/mmproj-SmolVLM-500M-Instruct-Q8_0.gguf -c 512 --host 0.0.0.0 --port 8080 --n-gpu-layers 0
```

Test the server with a simple request:

```bash
curl -X POST http://localhost:8080/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "SmolVLM-500M-Instruct-Q8_0", "messages": [{"role": "user", "content": "你好"}]}'
```

### 12. Create and Edit the Webcam Script
Create the `webcam_smolvlm.py` script in the project directory:

```bash
cd ~/smolvlm_project/smolvlm-realtime-webcam
nano webcam_smolvlm.py
```

Paste the following code into `webcam_smolvlm.py`:

```python
import cv2
import requests
import json
import base64
import time

# Initialize Webcam
cap = cv2.VideoCapture(0)  # Use /dev/video0, adjust if needed
cap.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 160)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 120)
cap.set(cv2.CAP_PROP_FPS, 10)

# llama.cpp server URL
LLAMA_SERVER_URL = "http://localhost:8080/v1/chat/completions"

frame_count = 0
while True:
    ret, frame = cap.read()
    if not ret:
        print("Unable to read from webcam")
        break

    frame_count += 1
    if frame_count % 10 == 0:
        # Convert image to JPEG and encode to base64
        _, buffer = cv2.imencode('.jpg', frame, [int(cv2.IMWRITE_JPEG_QUALITY), 80])
        image_base64 = base64.b64encode(buffer).decode('utf-8')

        # Build request payload
        payload = {
            "model": "SmolVLM-500M-Instruct-Q8_0",
            "messages": [
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": "Please briefly describe this image"},
                        {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{image_base64}"}}
                    ]
                }
            ],
            "max_tokens": 100,
            "stream": False
        }

        # Send request to llama.cpp server
        try:
            response = requests.post(
                LLAMA_SERVER_URL,
                json=payload,
                headers={"Content-Type": "application/json"},
                timeout=30
            )
            response.raise_for_status()
            response_json = response.json()
            if "choices" in response_json and response_json["choices"]:
                description = response_json["choices"][0]["message"]["content"]
                print("Model output:", description)
                with open("output.txt", "a") as f:
                    f.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')}: {description}\n")
            else:
                print("No valid response:", response_json)
        except requests.exceptions.RequestException as e:
            print(f"Request failed: {e}, Response: {response.text if 'response' in locals() else 'No response'}")
        except (KeyError, IndexError) as e:
            print(f"Response parsing failed: {e}, Response: {response.text if 'response' in locals() else 'No response'}")

    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

    time.sleep(0.2)

# Release resources
cap.release()
cv2.destroyAllWindows()
```

Save and exit the editor.

### 13. Run the Webcam Script
Ensure the llama.cpp server is running in another terminal, then execute the script:

```bash
python3 webcam_smolvlm.py
```

### 14. Troubleshooting
- **Webcam Issues**: If you see errors like `Unable to read from webcam` or GStreamer warnings, ensure the webcam is properly connected and `/dev/video0` is the correct device. Adjust the device index in `cv2.VideoCapture(0)` if needed.
- **Server Issues**: If the script fails to connect to the llama.cpp server, verify that the server is running and accessible at `http://localhost:8080`.
- **Performance**: The script processes every 10th frame to reduce load. Adjust `frame_count % 10` or `time.sleep(0.2)` for different performance needs.
- **KeyboardInterrupt**: Press `Ctrl+C` to stop the script. Ensure resources are released properly by the `cap.release()` and `cv2.destroyAllWindows()` calls.

### Expected Output
The script captures webcam frames, sends them to the SmolVLM model via the llama.cpp server, and logs descriptions to `output.txt`. Example output:

```
Model output: A young Asian man with short black hair is making a peace sign gesture.
```

The description is saved with a timestamp in `output.txt`.

## Notes
- The script assumes the webcam supports MJPG format and a resolution of 160x120 at 10 FPS. Adjust these settings based on your webcam's capabilities.
- The llama.cpp server runs without GPU acceleration (`--n-gpu-layers 0`). If you have a compatible GPU, you can adjust this parameter.
- Ensure sufficient disk space for the model files (~500 MB each).
# Raspberry Pi SmolVLM 即時網路攝影機設定筆記

本筆記提供在 Raspberry Pi 上設定並運行 [SmolVLM 即時網路攝影機項目](https://github.com/ngxson/smolvlm-realtime-webcam) 的詳細步驟。該項目使用 USB 網路攝影機捕捉影像，並透過 SmolVLM-500M-Instruct 模型生成圖像描述，適用於即時影像處理應用。

## 前置條件
- **硬體**：Raspberry Pi（建議 4 或更高型號），已安裝 Raspberry Pi OS（基於 Linux）。
- **設備**：USB 網路攝影機，連接到 Raspberry Pi。
- **網路**：穩定的互聯網連線，用於下載套件和模型檔案。
- **儲存空間**：至少 2GB 可用空間，用於模型檔案和依賴項。

## 步驟說明

### 1. 更新系統
確保系統套件為最新版本，以避免相容性問題。

```bash
sudo apt update
sudo apt full-upgrade -y
```

### 2. 安裝 Python 和 Git
安裝 Python 3、pip 和 Git，用於管理項目依賴和克隆儲存庫。

```bash
sudo apt install -y python3 python3-pip git
```

驗證 Python 版本（應為 3.x）：

```bash
python3 --version
```

### 3. 安裝網路攝影機依賴項
安裝 `fswebcam` 和 `v4l-utils`，以支援網路攝影機操作。

```bash
sudo apt install -y fswebcam v4l-utils
```

### 4. 驗證網路攝影機
檢查網路攝影機是否正確連接到 Raspberry Pi：

```bash
lsusb
```

列出可用視訊設備（通常為 `/dev/video0`）：

```bash
ls /dev/video*
```

測試網路攝影機，拍攝一張 640x480 的圖像：

```bash
fswebcam -d /dev/video0 -r 640x480 test.jpg
```

檢查網路攝影機支援的格式和分辨率：

```bash
v4l2-ctl --list-formats-ext
```

> **注意**：若 `test.jpg` 無法生成，確認 `/dev/video0` 是否為正確設備，或檢查攝影機連接。

### 5. 建立項目目錄
創建專用目錄存放項目檔案：

```bash
mkdir -p ~/smolvlm_project
cd ~/smolvlm_project
```

### 6. 克隆 SmolVLM 網路攝影機儲存庫
下載即時網路攝影機項目程式碼：

```bash
git clone https://github.com/ngxson/smolvlm-realtime-webcam.git
cd smolvlm-realtime-webcam
```

### 7. 安裝 Python 依賴項
安裝圖像處理和 HTTP 請求所需的 Python 庫：

```bash
sudo apt install -y python3-opencv python3-requests python3-pil
```

驗證已安裝的庫版本：

```bash
python3 -c "import cv2; print(cv2.__version__)"
python3 -c "import requests; print(requests.__version__)"
python3 -c "from PIL import Image; print(Image.__version__)"
```

### 8. 安裝構建工具
安裝構建 llama.cpp 所需的工具：

```bash
sudo apt install -y build-essential cmake
```

### 9. 克隆並構建 llama.cpp
克隆 [llama.cpp](https://github.com/ggerganov/llama.cpp) 儲存庫並構建，用於運行 SmolVLM 模型：

```bash
cd ~/smolvlm_project
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
```

安裝額外依賴項：

```bash
sudo apt update
sudo apt install -y libcurl4-openssl-dev
```

構建 llama.cpp：

```bash
rm -rf build
mkdir build
cd build
cmake ..
cmake --build . --config Release
```

驗證構建是否成功：

```bash
./bin/llama-cli --version
```

### 10. 下載 SmolVLM 模型檔案
下載 SmolVLM-500M-Instruct 模型及其多模態投影檔案：

```bash
wget https://huggingface.co/ggml-org/SmolVLM-500M-Instruct-GGUF/resolve/main/SmolVLM-500M-Instruct-Q8_0.gguf -O ~/smolvlm_project/SmolVLM-500M-Instruct-Q8_0.gguf
wget https://huggingface.co/ggml-org/SmolVLM-500M-Instruct-GGUF/resolve/main/mmproj-SmolVLM-500M-Instruct-Q8_0.gguf -O ~/smolvlm_project/mmproj-SmolVLM-500M-Instruct-Q8_0.gguf
```

驗證檔案是否下載成功：

```bash
ls -lh ~/smolvlm_project/SmolVLM-500M-Instruct-Q8_0.gguf
ls -lh ~/smolvlm_project/mmproj-SmolVLM-500M-Instruct-Q8_0.gguf
```

> **注意**：每個檔案約 500MB，確保有足夠儲存空間。

### 11. 啟動 llama.cpp 伺服器
運行 llama.cpp 伺服器以載入 SmolVLM 模型：

```bash
cd ~/smolvlm_project/llama.cpp/build/bin
./llama-server -m ~/smolvlm_project/SmolVLM-500M-Instruct-Q8_0.gguf --mmproj ~/smolvlm_project/mmproj-SmolVLM-500M-Instruct-Q8_0.gguf -c 512 --host 0.0.0.0 --port 8080 --n-gpu-layers 0
```

測試伺服器是否正常運行：

```bash
curl -X POST http://localhost:8080/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "SmolVLM-500M-Instruct-Q8_0", "messages": [{"role": "user", "content": "你好"}]}'
```

> **注意**：伺服器需在背景或另一終端機中持續運行。

### 12. 創建網路攝影機腳本
在項目目錄中創建 `webcam_smolvlm.py` 腳本：

```bash
cd ~/smolvlm_project/smolvlm-realtime-webcam
nano webcam_smolvlm.py
```

貼上以下程式碼：

```python
import cv2
import requests
import json
import base64
import time

# 初始化網路攝影機
cap = cv2.VideoCapture(0)  # 使用 /dev/video0，根據需要調整
cap.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 160)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 120)
cap.set(cv2.CAP_PROP_FPS, 10)

# llama.cpp 伺服器 URL
LLAMA_SERVER_URL = "http://localhost:8080/v1/chat/completions"

frame_count = 0
while True:
    ret, frame = cap.read()
    if not ret:
        print("無法從網路攝影機讀取影像")
        break

    frame_count += 1
    if frame_count % 10 == 0:
        # 將影像轉為 JPEG 並編碼為 base64
        _, buffer = cv2.imencode('.jpg', frame, [int(cv2.IMWRITE_JPEG_QUALITY), 80])
        image_base64 = base64.b64encode(buffer).decode('utf-8')

        # 構建請求 payload
        payload = {
            "model": "SmolVLM-500M-Instruct-Q8_0",
            "messages": [
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": "請簡要描述這張圖像"},
                        {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{image_base64}"}}
                    ]
                }
            ],
            "max_tokens": 100,
            "stream": False
        }

        # 發送請求到 llama.cpp 伺服器
        try:
            response = requests.post(
                LLAMA_SERVER_URL,
                json=payload,
                headers={"Content-Type": "application/json"},
                timeout=30
            )
            response.raise_for_status()
            response_json = response.json()
            if "choices" in response_json and response_json["choices"]:
                description = response_json["choices"][0]["message"]["content"]
                print("模型輸出：", description)
                with open("output.txt", "a") as f:
                    f.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')}: {description}\n")
            else:
                print("無有效回應：", response_json)
        except requests.exceptions.RequestException as e:
            print(f"請求失敗：{e}, 回應：{response.text if 'response' in locals() else '無回應'}")
        except (KeyError, IndexError) as e:
            print(f"解析回應失敗：{e}, 回應：{response.text if 'response' in locals() else '無回應'}")

    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

    time.sleep(0.2)

# 釋放資源
cap.release()
cv2.destroyAllWindows()
```

儲存並退出編輯器（`Ctrl+O`, `Enter`, `Ctrl+X`）。

### 13. 運行網路攝影機腳本
確保 llama.cpp 伺服器在另一終端機運行，然後執行腳本：

```bash
python3 webcam_smolvlm.py
```

### 14. 故障排除
- **網路攝影機問題**：
  - 若出現 `無法從網路攝影機讀取影像` 或 GStreamer 警告，檢查攝影機是否正確連接，確認 `/dev/video0` 是否為正確設備。
  - 調整 `cv2.VideoCapture(0)` 中的設備索引（例如，改為 `1` 或 `2`）。
- **伺服器問題**：
  - 若腳本無法連接到伺服器，確認 llama.cpp 伺服器正在運行，且 `http://localhost:8080` 可訪問。
  - 使用 `curl` 命令重新測試伺服器連線。
- **性能問題**：
  - 腳本每 10 幀處理一次（`frame_count % 10`），以降低 Raspberry Pi 的負載。可調整此值或 `time.sleep(0.2)` 來改變處理頻率。
- **中斷程式**：
  - 按 `Ctrl+C` 停止腳本，程式會自動釋放資源（`cap.release()` 和 `cv2.destroyAllWindows()`）。
- **GStreamer 警告**：
  - 警告如 `[ WARN:0@2.642] global ./modules/videoio/src/cap_gstreamer.cpp` 可忽略，僅影響日誌輸出，不影響功能。

### 預期輸出
腳本會即時捕捉網路攝影機影像，發送到 llama.cpp 伺服器進行處理，並將模型生成的描述記錄到 `output.txt`。示例輸出：

```
模型輸出：一位年輕的亞洲男子，短黑髮，正在做出和平手勢。
```

輸出將以時間戳記錄在 `output.txt`，例如：

```
2025-05-23 12:29:00: 一位年輕的亞洲男子，短黑髮，正在做出和平手勢。
```

## 注意事項
- **攝影機設置**：腳本預設使用 MJPG 格式，分辨率 160x120，幀率 10 FPS。根據攝影機性能，可在 `webcam_smolvlm.py` 中調整 `CAP_PROP_FRAME_WIDTH`、`CAP_PROP_FRAME_HEIGHT` 和 `CAP_PROP_FPS`。
- **GPU 加速**：本設定使用 CPU 運行（`--n-gpu-layers 0`）。若 Raspberry Pi 支援 GPU 加速，可調整此參數以提升性能。
- **儲存空間**：模型檔案（`SmolVLM-500M-Instruct-Q8_0.gguf` 和 `mmproj-SmolVLM-500M-Instruct-Q8_0.gguf`）各約 500MB，確保有足夠空間。
- **網路連線**：下載模型檔案和依賴項需要穩定網路，建議使用有線網路以加快速度。

## 後續步驟
- **自訂描述**：修改 `payload` 中的 `"text": "請簡要描述這張圖像"`，以改變模型的任務，例如要求更詳細的描述或特定物件辨識。
- **日誌管理**：檢查 `output.txt` 以檢視所有生成描述，或修改腳本以將輸出儲存到其他格式（如 JSON）。
- **效能優化**：若 Raspberry Pi 性能有限，可降低分辨率或增加 `time.sleep` 間隔。
