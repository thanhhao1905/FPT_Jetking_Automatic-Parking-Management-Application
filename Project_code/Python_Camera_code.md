````python
import cv2
import tkinter as tk
from tkinter import Label, Button, filedialog, messagebox
from PIL import Image, ImageTk
import requests
import serial
from datetime import datetime
import os
import csv

# API Key từ PlateRecognizer
API_KEY = "61a40b3ce649455bf23e98ae1ac91740449cf77e"
API_URL = "https://api.platerecognizer.com/v1/plate-reader/"


ALLOWED_PLATES = {plate.upper() for plate in ["59Y228150", "81E119844", "59Y254100", "55Y48649"]}


latest_plate_number = ""
last_sent_plate = None


LOG_DIR = r"E:\CHIPBANDAN\BKSTEST\output"
os.makedirs(LOG_DIR, exist_ok=True)

CSV_LOG_PATH = os.path.join(LOG_DIR, "log.csv")


try:
    ser = serial.Serial('COM4', 9600, timeout=1)
except serial.SerialException:
    ser = None
    print(" Không thể mở cổng COM4!")

def send_plate_to_uart(plate_number):
    global last_sent_plate
    if ser and ser.is_open:
        if plate_number != last_sent_plate:
            if plate_number in ALLOWED_PLATES:
                ser.write(b"1\n")
                print(" Gửi 1 qua COM4 (hợp lệ)")
            else:
                ser.write(b"0\n")
                print(f" Gửi biển số không hợp lệ: {plate_number}")
            last_sent_plate = plate_number
    else:
        print(" COM4 không khả dụng!")

def save_log_csv(plate_number, time_str):
   
    file_exists = os.path.isfile(CSV_LOG_PATH)
    with open(CSV_LOG_PATH, "a", newline='', encoding="utf-8") as csvfile:
        writer = csv.writer(csvfile)
        if not file_exists:
            writer.writerow(["Thoi gian Ra", "Bien so"])
        writer.writerow([time_str, plate_number])

def save_processed_image(frame):
    resized_frame = cv2.resize(frame, (400, 300))  # Đảm bảo ảnh được resize trước khi lưu
    time_str_file = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = os.path.join(LOG_DIR, f"{time_str_file}.jpg")
    cv2.imwrite(filename, frame)
    print(f" Đã lưu ảnh: {filename}")
    
def draw_text_on_frame(frame, plate_number, current_time):
    font = cv2.FONT_HERSHEY_SIMPLEX
    color = (0, 0, 255) 
    thickness = 2
    margin = 20


    height, width = frame.shape[:2]

    font_scale = max(0.3, min(width / 400 * 0.6, 1.0))  # từ 0.5 đến 2.0, tỷ lệ với width

    text1 = f"BS: {plate_number}"
    text2 = f"Time: {current_time}"


    y1 = margin + int(30 * font_scale)
    y2 = y1 + int(40 * font_scale)

    cv2.putText(frame, text1, (margin, y1), font, font_scale, color, thickness, cv2.LINE_AA)
    cv2.putText(frame, text2, (margin, y2), font, font_scale, color, thickness, cv2.LINE_AA)

def show_and_detect(image_path, frame=None):
    global latest_plate_number
    try:
        if frame is None:
            frame = cv2.imread(image_path)

        with open(image_path, "rb") as f:
            response = requests.post(API_URL,
                                     files={"upload": f},
                                     headers={"Authorization": f"Token {API_KEY}"})
            response.raise_for_status()
            data = response.json()

        if data.get("results"):
            plate_number = data["results"][0]["plate"].upper()
            latest_plate_number = plate_number

            current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

            draw_text_on_frame(frame, plate_number, current_time)

            img_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            img_pil = Image.fromarray(img_rgb).resize((400, 300))
            img_tk = ImageTk.PhotoImage(img_pil)

            captured_image_label.config(image=img_tk)
            captured_image_label.image = img_tk

            result_label.config(text=f"Biển số: {plate_number}")

            save_log_csv(plate_number, current_time)
            save_processed_image(frame) 
            send_plate_to_uart(plate_number)

        else:
            result_label.config(text="Không nhận diện được biển số!")
            messagebox.showwarning("Cảnh báo", "Không nhận diện được biển số! Vui lòng thử lại.")
            
            if frame is not None:
                img_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
                img_pil = Image.fromarray(img_rgb).resize((400, 300))
                img_tk = ImageTk.PhotoImage(img_pil)
                captured_image_label.config(image=img_tk)
                captured_image_label.image = img_tk

    except requests.RequestException as e:
        result_label.config(text="Lỗi kết nối API!")
        messagebox.showerror("Lỗi API", f"Lỗi khi gọi PlateRecognizer:\n{e}")

    except Exception as e:
        result_label.config(text="Lỗi xử lý ảnh!")
        messagebox.showerror("Lỗi", f"Lỗi không xác định:\n{e}")


def update_camera_preview():
    ret, frame = cap.read()
    if ret:
        height, width, _ = frame.shape
        top_left = (int(width * 0.2), int(height * 0.4))
        bottom_right = (int(width * 0.8), int(height * 0.85))
        cv2.rectangle(frame, top_left, bottom_right, (0, 255, 0), 2)

        if latest_plate_number:
            cv2.putText(frame,
                        f"BS: {latest_plate_number}",
                        (top_left[0], top_left[1] - 10),
                        cv2.FONT_HERSHEY_SIMPLEX,
                        0.8,
                        (0, 0, 255),
                        2,
                        cv2.LINE_AA)

        img_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        img_pil = Image.fromarray(img_rgb).resize((400, 300))
        img_tk = ImageTk.PhotoImage(img_pil)
        live_camera_label.config(image=img_tk)
        live_camera_label.image = img_tk

    root.after(50, update_camera_preview)
def capture_image():
    ret, frame = cap.read()
    if ret:
        height, width, _ = frame.shape
        top_left = (int(width * 0.2), int(height * 0.4))
        bottom_right = (int(width * 0.8), int(height * 0.85))
        roi = frame[top_left[1]:bottom_right[1], top_left[0]:bottom_right[0]]
        cv2.imwrite("plate.jpg", roi)
        show_and_detect("plate.jpg", roi)

def process_static_image():
    file_path = filedialog.askopenfilename(filetypes=[("Image files", "*.jpg;*.jpeg;*.png")])
    if file_path:
        show_and_detect(file_path)

def quit_app():
    if cap: cap.release()
    if ser and ser.is_open: ser.close()
    root.destroy()

root = tk.Tk()
root.title("Nhận diện biển số xe")

capture_button = Button(root, text=" Chụp Ảnh", command=capture_image, font=("Arial", 14))
capture_button.grid(row=0, column=0, padx=10, pady=10)

select_image_button = Button(root, text=" Chọn Ảnh", command=process_static_image, font=("Arial", 14))
select_image_button.grid(row=0, column=1, padx=10, pady=10)

exit_button = Button(root, text=" Thoát", command=quit_app, font=("Arial", 14), fg="white", bg="red")
exit_button.grid(row=0, column=2, padx=10, pady=10)

result_label = Label(root, text="Biển số: ...", font=("Arial", 14))
result_label.grid(row=1, column=0, columnspan=3, padx=10, pady=5)


live_camera_label = Label(root)
live_camera_label.grid(row=3, column=0, columnspan=3, padx=10, pady=10)

captured_image_label = Label(root)
captured_image_label.grid(row=4, column=0, columnspan=3, padx=10, pady=10)

cap = cv2.VideoCapture(0)
update_camera_preview()

root.bind("<Return>", lambda event: capture_image()) # nhấn Enter chụp ảnh
root.mainloop()

if cap: cap.release()
if ser and ser.is_open: ser.close()
