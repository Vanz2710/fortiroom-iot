# FORTIROOM: IoT Smart Room Booking System

> **Smart. Secure. Seamless.** Final Year Project, HELP University (group project)

FORTIROOM is an IoT smart room booking and access-control system. Users book rooms (pods) through a web app. When they arrive, an ESP32-CAM captures their face, the backend verifies it against their profile photo using a face-recognition service, and if the face matches an active booking the door unlocks. An ESP32 controller in each room monitors temperature, humidity, air quality and motion, controls the lights and fan, and reports to a staff dashboard in real time.

---

## Features

**For users**
- Sign up, log in and reset passwords (Supabase Auth), with email and username availability checks
- Book rooms, and add a secondary user to a booking
- Booking confirmation emails
- Face-verified check-in at the door using the ESP32-CAM
- Profile management, including uploading an avatar that serves as the reference face image
- Pay penalties online with Stripe Checkout (e.g. for late checkout or motion detected after checkout)
- Request account deletion

**For staff**
- Dashboard with live room (pod) metrics: temperature, humidity, air quality, motion, door, light and fan state
- Remote unlock, light/fan control and checkout for each pod
- Manage users, access rules and penalty rates
- Review account deletion requests
- Analytics page with a printable report

**On the devices**
- ESP32 room controller with a DHT22 (temperature/humidity), MQ135 (air quality/smoke), PIR motion sensor, door-lock relay, light and fan outputs, RTC and OLED status display
- ESP32-CAM door camera with a capture button that triggers face verification
- Wi-Fi captive-portal setup, mDNS, OTA firmware updates and a telnet/web log viewer
- Devices register themselves with the backend so the server always knows their current IP

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | PHP-rendered pages, HTML/CSS/JavaScript, Tailwind CSS, Bootstrap, jQuery, Morris.js charts, DataTables, Font Awesome |
| Backend | PHP 8 REST-style JSON endpoints (cURL, GD) |
| Database / Auth / Storage | Supabase (PostgreSQL, Supabase Auth, Supabase Storage), using `supabase-js` v2 in the browser and the REST API from PHP |
| Face recognition | [CompreFace](https://github.com/exadel-inc/CompreFace) verification API, self-hosted on a Google Cloud Platform VM |
| Payments | Stripe Checkout (`stripe/stripe-php`) |
| Email | PHP `mail()` (HTML booking confirmations) |
| IoT hardware | ESP32 (room controller), ESP32-CAM (door camera), Arduino framework |
| Firmware libraries | `WiFi`, `WebServer`, `HTTPClient`, `Preferences`, `DNSServer`, `ESPmDNS`, `ArduinoOTA`, `esp_camera`, `DHT`, `RTClib`, `Adafruit_GFX`, `Adafruit_SH110X` |

---

## System Architecture

```
 ┌────────────────────┐        HTTPS          ┌───────────────────────────┐
 │  Web app (browser) │ ───────────────────▶  │   PHP backend (REST APIs) │
 │  User + Staff UI   │ ◀───────────────────  │   login, booking, staff,  │
 └─────────┬──────────┘                       │   pod control, face API   │
           │ supabase-js (auth)               └──┬─────────┬─────────┬────┘
           ▼                                     │         │         │
 ┌────────────────────┐   REST / Storage         │         │         │ HTTP (LAN)
 │      Supabase      │ ◀────────────────────────┘         │         ▼
 │ PostgreSQL · Auth  │                                    │  ┌──────────────────┐
 │ Storage (avatars)  │                                    │  │ ESP32 room ctrl. │
 └────────────────────┘                                    │  │ DHT22 · MQ135 ·  │
                                                           │  │ PIR · door relay │
 ┌────────────────────┐   verification API                 │  │ light · fan · OLED│
 │ CompreFace on GCP  │ ◀──────────────────────────────────┘  └────────▲─────────┘
 └────────────────────┘                                                │ button / unlock
                                                           ┌───────────┴──────────┐
                                                           │  ESP32-CAM (door)    │
                                                           └──────────────────────┘
```

**Door access flow**
1. The user presses the door button, and the ESP32-CAM captures an image.
2. The image is sent to `face_verify_api.php`, which fetches the user's avatar from Supabase Storage and calls CompreFace's verification API.
3. If the similarity is above the configured threshold and the user has an active booking for that pod, the backend tells the room controller to unlock (`/unlock`).
4. The room controller polls `env_booking_status.php` for booking state and reports motion after checkout to `checkout_motion_penalty.php`, which can create a penalty.

**Main database tables:** `bookings`, `pods`, `penalties`, `penalty_rates`, `access_rules`, `account_deletion_requests`

---

## Results

From project testing:
- **96% facial recognition accuracy**
- **3–5 seconds** from face capture to door unlock

---

## My Role

This was a **team project**. My responsibilities were:
- **Designing the backend architecture and REST APIs**
- **Integrating the application with the devices**, connecting the web app and backend to the ESP32 room controller and ESP32-CAM

---

## Project Structure

```
├── index.php, login.php, reset.php    # Public landing, auth, password reset
├── face_verify_api.php                # Face verification (CompreFace) endpoint
├── face_verify_test.php               # Face verification test page
├── esp32_cam_registry.php             # ESP32-CAM self-registration
├── esp32_env_registry.php             # Room controller self-registration
├── env_booking_status.php             # Booking status polled by the room controller
├── checkout_motion_penalty.php        # Motion-after-checkout penalty reporting
├── user/                              # User dashboard, bookings, profile, Stripe, emails
├── staff/                             # Staff dashboard, users, pods, analytics, penalties
│   └── api/                           # Pod control: unlock, light, fan, metrics, checkout
├── envcode/
│   ├── envcode.ino                    # ESP32 room controller firmware
│   └── esp32_cam_capture_test/        # ESP32-CAM firmware
├── assets/, css/, js/, fonts/, images/
├── composer.json
└── .env.example
```

---

## Running Locally

### Prerequisites
- PHP 8.0+ with the `curl` and `gd` extensions (e.g. XAMPP)
- [Composer](https://getcomposer.org/)
- A [Supabase](https://supabase.com/) project (database, auth and a storage bucket for avatars)
- A [Stripe](https://stripe.com/) account (test mode)
- A running [CompreFace](https://github.com/exadel-inc/CompreFace) instance with a **verification** service API key
- *(Optional, for the full IoT setup)* An ESP32 dev board with the sensors listed above, an ESP32-CAM and the Arduino IDE

### 1. Backend and web app
```bash
git clone https://github.com/Vanz2710/fortiroom-iot.git
cd fortiroom-iot
composer install
cp .env.example .env      # then fill in your own values
```
Put the project in your web server root (e.g. `C:/xampp/htdocs/Fortiroom`) and open `http://localhost/Fortiroom/`.

In Supabase, create the tables listed above and an `avatars` storage bucket (or whatever name you set in `AVATARS_BUCKET`).

### 2. Environment variables
All configuration lives in `.env`. See [`.env.example`](.env.example) for the full list:

| Variable | Purpose |
|---|---|
| `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_KEY` | Supabase project and keys |
| `AVATARS_BUCKET`, `STORAGE_ENDPOINT` | Supabase Storage for profile/face images |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Staff/admin account |
| `STRIPE_SECRET_KEY` | Stripe penalty payments |
| `ESP_ENV_DEVICE_ID`, `ESP_SENSORS_PATH`, `ESP_HTTP_TIMEOUT_SEC` | Room controller connection |
| `COMPRE_FACE_BASE_URL`, `COMPRE_FACE_API_KEY`, `COMPRE_FACE_SIMILARITY_THRESHOLD` | Face verification |

### 3. ESP32 firmware (optional)
1. Open `envcode/envcode.ino` (room controller) and `envcode/esp32_cam_capture_test/esp32_cam_capture_test.ino` (camera) in the Arduino IDE.
2. Set `HARDCODED_WIFI_SSID`, `HARDCODED_WIFI_PASS` and `DEFAULT_SERVER_BASE_URL` (the IP of the machine running the PHP backend). Alternatively, set `USE_HARDCODED_WIFI = false` and use the built-in Wi-Fi setup portal.
3. Install the libraries listed in the tech stack and flash each board. Wiring for the room controller is documented at the top of `envcode.ino`.
4. The devices register themselves with the backend once they're online.

> **Never commit `.env` or real Wi-Fi credentials.** Both are excluded by `.gitignore`.

---

## Screenshots

> _Screenshots coming soon._

| Landing page | Login |
|---|---|
| ![Landing page](docs/screenshots/landing.png) | ![Login](docs/screenshots/login.png) |

| User dashboard | Booking |
|---|---|
| ![User dashboard](docs/screenshots/user-dashboard.png) | ![Booking](docs/screenshots/booking.png) |

| Staff dashboard | Pod monitoring |
|---|---|
| ![Staff dashboard](docs/screenshots/staff-dashboard.png) | ![Pod monitoring](docs/screenshots/pods.png) |

| Hardware setup | Face verification |
|---|---|
| ![Hardware](docs/screenshots/hardware.jpg) | ![Face verification](docs/screenshots/face-verify.png) |

---

## Acknowledgements

Developed as a Final Year Project at **HELP University** by a team of students.
