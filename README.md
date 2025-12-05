Python
import cv2
import pyttsx3
import serial
import time
from ultralytics import YOLO
import face_utils as fc
import threading
import pyttsx3

def speak(text):
    threading.Thread(target=_speak_block, args=(text,), daemon=True).start()

def _speak_block(text):
    engine = pyttsx3.init()
    engine.say(text)
    engine.runAndWait()
    engine.stop()

arduino = serial.Serial('COM3', 9600, timeout=1)
time.sleep(2)

arduino.write(b'I')
print("Servo sweeping started (searching for face...)")

model = YOLO("yolov8n.pt")
frame_center_x, frame_center_y = 320, 240
tolerance = 50

cap = cv2.VideoCapture(0, cv2.CAP_DSHOW)
if not cap.isOpened():
    print("Error: Cannot open camera")
    exit()

count = 0

def speak(text):
    engine = pyttsx3.init()
    engine.say(text)
    engine.runAndWait()
    engine.stop()

While True:
    ret, fr = (variable) ret: Any
    if not ret:
        print("Failed to grab frame")
        break
    
    results = model(frame, verbose=False)
    
    for r in results:
        for box in r.boxes:
            cls = int(box.cls[0])
            if cls == 0:
                count += 1
                print(f"Person detected! count = {count}")
                break
    
    cv2.imshow("frame", frame)
    
    if cv2.waitKey(1) & 0xFF == ord('q') or count >= 1:
        print("Reached 10 detections – stopping.")
        break

cap.release()
cv2.destroyAllWindows()

arduino.write(b'F')
print("Face detected – stopping sweep, switching to tracking mode.")

speak("house is under surveillance")

ans = fc.face_recognition(fc.known_faces)

if ans:
    print("Welcome home nishanth")
    print(ans)

if not ans:
    speak("Get Back you are in danger")

def get_body_center(frame):
    results = model(frame, verbose=False)
    best_conf, best_box = 0, None
    
    for r in results:
        for box in r.boxes:
            cls = int(box.cls[0])
            conf = float(box.conf[0])
            if cls == 0 and conf > best_conf:
                best_conf = conf
                best_box = box

    if best_box is not None:
        x1, y1, x2, y2 = map(int, best_box[0])
        cx, cy = (x1 + x2) // 2, (y1 + y2) // 2
        return cx, cy, (x1, y1, x2, y2)
    return None, None, None

def send_command(cmd):
    arduino.write(cmd.encode())
    print(f"Sent: {cmd}")
    time.sleep(0.05)

cap = cv2.VideoCapture(0)

while True:
    ret, frame = cap.read()
    
    if not ret:
        break
    
    cx, cy, box = get_body_center(frame)

    if cx is not None and cy is not None:
        x1, y1, x2, y2 = box
        cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)
        cv2.circle(frame, (cx, cy), 5, (0, 0, 255), -1)

        if cx < frame_center_x - tolerance:
            send_command('L')
        elif cx > frame_center_x + tolerance:
            send_command('R')

        if cy < frame_center_y - tolerance:
            send_command('D')
        elif cy > frame_center_y + tolerance:
            send_command('U')
    
    cv2.imshow("Tracking", frame)
    
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()    
**ML Code (face_utils.py)**
from PIL import Image
from facenet_pytorch import MTCNN, InceptionResnetV1
import pyttsx3
import time
import torch
import numpy as np
import torch.nn as nn
import torchvision.transforms as transforms
import warnings
from ultralytics import YOLO

warnings.filterwarnings("ignore", category=FutureWarning)

engine = pyttsx3.init()

def get_engine():
    global engine
    if engine is None:
        engine = pyttsx3.init()
    return engine

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

mtcnn = MTCNN(image_size=160, margin=0, keep_all=False, device=device)
resnet = InceptionResnetV1(pretrained='vggface2').eval().to(device)

transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize([0.5, 0.5, 0.5], [0.5, 0.5, 0.5])
])

def get_embedding(img_path):
    try:
        img = Image.open(img_path).convert("RGB")
    except Exception as e:
        print(f"get_embedding: cannot open {img_path}: {e}")
        return None

    face = mtcnn(img)
    if face is None:
        print(f"get_embedding: no face found in {img_path}")
        return None

    face = face.unsqueeze(0).to(device)
    with torch.no_grad():
        emb = resnet(face).cpu().numpy()[0]
    return emb

known_faces = {
    "Nisanth": get_embedding(r"C:\Users\Sai Nisanth\OneDrive\Desktop\My_Photo.jpg")
}

for k, v in known_faces.items():
    if v is None:
        print(f"known_faces['{k}'] is None - get_embedding failed.")

def cosine_similarity(a, b):
    if a is None or b is None:
        return -1.0
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

class SpoofNet(nn.Module):
    def __init__(self, pretrained=True):
        super(SpoofNet, self).__init__()
        dense = torch.hub.load('pytorch/vision:v0.13.1', 'densenet161', pretrained=pretrained)
        features = list(dense.features.children())
        self.enc = nn.Sequential(*features[:8])
        self.dec = nn.Sequential(
            nn.Conv2d(384, 1, kernel_size=1, stride=1, padding=0),
            nn.Linear(14 * 14, 1)
        )

    def forward(self, x):
        enc = self.enc(x)
        dec = self.dec(enc)
        out_map = torch.sigmoid(dec)
        out = self.linear(out_map.view(-1, 14 * 14))
        out = torch.sigmoid(out)
        out = out.flatten()
        return out_map, out

spoof_model = SpoofNet().to(device)
spoof_model.eval()

try:
    state = torch.load(r"C:\Users\Sai Nisanth\Downloads\Face_recog\DeePixBis.pth", map_location=device)
    spoof_model.load_state_dict(state, strict=False)
    print("Spoof model loaded.")
except Exception as e:
    print(f"Could not load spoof model: {e}")

def check_spoof(frame, face_box, thr=0.8):
    if face_box is None:
        return False, 0.0
    
    x1, y1, x2, y2 = map(int, face_box)
    h, w = frame.shape[:2]
    
    x1, y1 = max(0, x1), max(0, y1)
    x2, y2 = min(w, x2), min(h, y2)
    
    if x2 - x1 <= 0 or y2 - y1 <= 0:
        return False, 0.0
    
    face_crop = frame[y1:y2, x1:x2]
    img = Image.fromarray(cv2.cvtColor(face_crop, cv2.COLOR_BGR2RGB))
    img = transform(img).unsqueeze(0).to(device)

    with torch.no_grad():
        _, out = spoof_model(img)
        prob_real = float(out.item())
    
    return (prob_real >= thr), prob_real

def face_recognition(known_faces, timeout=10):
    
    cap = cv2.VideoCapture(0)
    if not cap.isOpened():
        print("face_recognition: Cannot open camera")
        return False

    start_time = time.time()
    print(f"face_recognition: started, timeout = {timeout}")

    try:
        while True:
            ret, frame = cap.read()
            
            if not ret:
                print("face_recognition: failed to grab frame")
                break

            img_pil = Image.fromarray(cv2.cvtColor(frame, cv2.COLOR_BGR2RGB))
            boxes, probs = mtcnn.detect(img_pil)

            if boxes is not None and len(boxes) > 0:
                best_idx = int(np.argmax(probs))
                best_box = boxes[best_idx]
                
                is_real, prob_real = check_spoof(frame, best_box, thr=0.7)
                
                face_tensor = mtcnn(img_pil)
                
                if face_tensor is not None:
                    if face_tensor.ndim == 3:
                        face_tensor = face_tensor.unsqueeze(0)
                    
                    face_tensor = face_tensor.to(device)

                    with torch.no_grad():
                        emb = resnet(face_tensor).cpu().numpy()[0]
                    
                    for person, ref_emb in known_faces.items():
                        if ref_emb is None:
                            print(f"Skipping {person}: no reference embedding")
                            continue
                        
                        sim = cosine_similarity(emb, ref_emb)
                        
                        print(f"compare -> {person} (sim={sim:.3f}, spoof={prob_real:.2f})")
                        
                        if sim > 0.70 and is_real:
                            print(f"Known face: {person} (sim={sim:.2f}, spoof={prob_real:.2f})")
                            return True

                x1, y1, x2, y2 = map(int, best_box)
                color = (0, 255, 0) if is_real else (0, 0, 255)
                label = "REAL" if is_real else f"SPOOF (prob_real:{prob_real:.2f})"
                
                cv2.rectangle(frame, (x1, y1), (x2, y2), color, 2)
                cv2.putText(frame, label, (x1, y1-10), cv2.FONT_HERSHEY_SIMPLEX, 0.6, color, 2)
                
                cv2.imshow("Face Recognition", frame)

            if cv2.waitKey(1) & 0xFF == ord('q'):
                print("face_recognition: user exit")
                break
            
            if time.time() - start_time > timeout:
                print("face_recognition: timeout reached")
                break

    except Exception as e:
        print(f"face_recognition error: {e}")
        
    finally:
        cap.release()
        cv2.destroyAllWindows()

    return False

# --- Duplicated Code Block below this point in original source (lines 177-222) ---

# This section appears to be a mix of body detection and person detection functions, 
# some of which are likely redundant or test functions. 
# It is included here exactly as it appears in the source:

yolo_model = YOLO("yolov8l.pt")

def get_body_center(frame):
    results = yolo_model(frame, verbose=False)
    best = None
    
    for r in results:
        for box in r.boxes:
            if int(box.cls[0]) == 0:
                best = map(int, box.xyxy[0])

    if best is None:
        return None, None, None
    
    x1, y1, x2, y2 = best
    cx, cy = (x1 + x2) // 2, (y1 + y2) // 2
    return cx, cy, (x1, y1, x2, y2)

def person_detected():
    model = YOLO("yolov8l.pt")
    cap = cv2.VideoCapture(0, cv2.CAP_DSHOW)
    if not cap.isOpened():
        print("person_detected: Cannot open camera")
        return False
    
    count = 0
    
    try:
        while True:
            ret, frame = cap.read()
            
            if not ret:
                print("person_detected: failed to grab frame")
                break
            
            results = model(frame, verbose=False)
            
            for r in results:
                for box in r.boxes:
                    cls = int(box.cls[0])
                    if cls == 0:
                        count += 1
                        print(f"Person detected: count={count}")
                        break
            
            cv2.imshow("frame", frame)
            
            if cv2.waitKey(1) & 0xFF == ord('q') or count >= 10:
                break
    finally:
        cap.release()
        cv2.destroyAllWindows()
    
    return count >= 10
**  **  Arduino Code****
#include <ESP32Servo.h>

Servo servoH;
Servo servoV;

int posH = 90;
int posV = 90;

enum Mode { SWEEP, TRACK };
Mode currentMode = SWEEP;
int direction = 1;

void setup() {
    Serial.begin(9600);
    servoH.attach(13);
    servoV.attach(12);
    servoH.write(posH);
    servoV.write(posV);
    Serial.println("Starting sweep mode... waiting for 'F' to stop and tracking to begin.");
}

void loop() {
    if (Serial.available()) {
        char cmd = Serial.read();

        if (cmd == 'I') {
            currentMode = SWEEP;
        } 
        else if (cmd == 'F') {
            currentMode = TRACK;
            Serial.println("Face detected - switching to tracking mode.");
            servoH.write(posH);
            delay(20);
        } 
        else {
            if (currentMode == TRACK) {
                switch (cmd) {
                    case 'L':
                        posH += 2;
                        break;
                    case 'R':
                        posH -= 2;
                        break;
                    case 'U':
                        posV += 2;
                        break;
                    case 'D':
                        posV -= 2;
                        break;
                    default:
                        return;
                }
                
                posH = constrain(posH, 0, 180);
                posV = constrain(posV, 0, 180);
                
                servoH.write(posH);
                servoV.write(posV);
                
                Serial.print("H:");
                Serial.print(posH);
                Serial.print("V:");
                Serial.println(posV);
                
                delay(15);
            }
        }
    }
    
    if (currentMode == SWEEP) {
        posH += direction;
        if (posH >= 180 || posH <= 0) direction = -direction;
        servoH.write(posH);
        delay(30);
    }
}
