# Final-Project-SIC-4-Kelompk-3
Kebutuhan utama pengguna, yaitu perlindungan atap agar tidak terganggu oleh cuaca ekstrem, kenyamanan selama beraktivitas, dan kemampuan untuk mengendalikan atap sesuai kebutuhan. Proyek Drizzel Detector akan memberikan solusi bagi berbagai kelompok pengguna dengan berbagai kebutuhan.


Coding Drizzle Detector

from datetime import datetime
import time
import RPi.GPIO as GPIO
import time
import requests
import mysql.connector
from ubidots import ApiClient

# Konfigurasi GPIO
pin_hujan = 18
relay1_pin = 23
relay2_pin = 24
GPIO.setmode(GPIO.BCM)
GPIO.setup(pin_hujan, GPIO.IN)
GPIO.setup(relay1_pin, GPIO.OUT)
GPIO.setup(relay2_pin, GPIO.OUT)

# Mematikan kedua relay saat aplikasi pertama kali dijalankan
GPIO.output(relay1_pin, GPIO.HIGH)
GPIO.output(relay2_pin, GPIO.HIGH)

# Inisialisasi koneksi ke Ubidots
ubidots = ApiClient(token="BBFF-2KOQukGazgRJI8imHFkkFRuh7U84rC")

# Dapatkan referensi ke variabel di Ubidots
rain_status_variable = ubidots.get_variable("64e3316a8408ac5262921f40")
relay1 = ubidots.get_variable("64eb1946a966070011e699b3")
relay2 = ubidots.get_variable("64eb19912fb97b000bd38e3b")

    
# Fungsi untuk mendeteksi status hujan
def detect_rain():
    return not GPIO.input(pin_hujan)

# Fungsi untuk mengaktifkan relay selama durasi waktu
def activate_relay(pin, duration):
    GPIO.output(pin, GPIO.HIGH)
    time.sleep(duration)
    GPIO.output(pin, GPIO.LOW)

# Fungsi untuk mengirim pesan ke Telegram menggunakan API Requests
def send_telegram_message(message):
    try:
        telegram_token = "6697065784:AAHnXHbnqRX_qGtGWulKELqHLz2WDKAUJY4"
        chat_id = "-1001983002002"
        url = f"https://api.telegram.org/bot{telegram_token}/sendMessage"
        params = {
            "chat_id": chat_id,
            "text": message
        }
        response = requests.post(url, data=params)
        if response.status_code == 200:
            print("Pesan berhasil dikirim ke Telegram.")
        else:
            print("Gagal mengirim pesan ke Telegram. Kode status:", response.status_code)
    except requests.exceptions.RequestException as e:
        print("Terjadi kesalahan saat mengirim pesan:", str(e))

relay1_state=False
relay1_count = 0

relay2_state=False
relay2_count = 0

try:
    last_rain_status = detect_rain()
    
    while True:
        current_rain_status = detect_rain()
        data_relay_1=relay1.get_values(1)
        data_relay_2=relay2.get_values(1)
        
        if relay1_count > 0:
            GPIO.output(23, GPIO.HIGH)
            relay1_count = relay1_count-1
            send_telegram_message("Atap Tertutup")  # Kirim laporan ke Telegram
            
        else:
            GPIO.output(23, GPIO.LOW)
            
        if relay2_count > 0:
            GPIO.output(24, GPIO.HIGH)
            relay2_count = relay2_count-1                
            send_telegram_message("Atap Terbuka")  # Kirim laporan ke Telegram
            
        else:
            GPIO.output(24, GPIO.LOW)
                    
        if data_relay_1[0]['value']:
            if relay1_state == False:
                relay1_state=True
                relay1_count=1
        else :
            relay1_state =False
            relay1_count =0
            
        if data_relay_2[0]['value']:
            if relay2_state == False:
                relay2_state=True
                relay2_count=1
        else :
            relay2_state =False
            relay2_count =0
            
        if current_rain_status != last_rain_status:
            last_rain_status = current_rain_status
            
            if current_rain_status:  # Jika hujan
                print("Hujan terdeteksi")
                send_telegram_message("Hujan terdeteksi!")  # Kirim laporan ke Telegram
                send_telegram_message("Atap Tertutup")  # Kirim laporan ke Telegram
                rain_status_variable.save_value({"value": 1})  # Kirim data ke Ubidots
                relay1_count =1
                
            else:  # Jika tidak hujan
                print("Hujan tidak terdeteksi")
                send_telegram_message("Tidak ada hujan.")  # Kirim laporan ke Telegram
                rain_status_variable.save_value({"value": 0})  # Kirim data ke Ubidots
                relay2_count =1
        
        time.sleep(0.3)  # Periksa kondisi setiap setengah detik
         
except KeyboardInterrupt:
    GPIO.cleanup()
