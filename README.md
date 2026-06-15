import json
import os
import re
import time

from plyer import notification
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager

# ==========================
# SETTINGS
# ==========================

ORIGIN = "MSY"          # New Orleans
DESTINATION = "KTM"     # Kathmandu
DEPARTURE_DATE = "2026-11-10"

PRICE_FILE = "qatar_msy_ktm_price.json"

ALERT_THRESHOLD = 50
TARGET_PRICE = 1300

AIRLINE_KEYWORDS = ["Qatar", "Qatar Airways"]

# ==========================
# DESKTOP NOTIFICATION
# ==========================

def send_notification(title, message):
    try:
        notification.notify(
            title=title,
            message=message,
            timeout=15
        )
    except Exception as e:
        print("Notification error:", e)

# ==========================
# PRICE STORAGE
# ==========================

def load_prices():
    if os.path.exists(PRICE_FILE):
        with open(PRICE_FILE, "r") as f:
            return json.load(f)
    return {}

def save_prices(prices):
    with open(PRICE_FILE, "w") as f:
        json.dump(prices, f, indent=2)

# ==========================
# GOOGLE FLIGHTS CHECKER
# ==========================

def get_qatar_price(driver):

    url = (
        f"https://www.google.com/travel/flights?"
        f"hl=en#flt={ORIGIN}.{DESTINATION}.{DEPARTURE_DATE};"
        f"c:USD;e:1;sd:1;t:f"
    )

    print(f"Checking Qatar Airways: {ORIGIN} → {DESTINATION}")
    print(url)

    driver.get(url)
    time.sleep(12)

    html = driver.page_source

    if not any(keyword in html for keyword in AIRLINE_KEYWORDS):
        print("Qatar Airways not found on this page.")
        return None

    # Try to find dollar amounts near Qatar text
    qatar_sections = re.findall(
        r"Qatar.{0,1500}?\$([0-9,]+)|\$([0-9,]+).{0,1500}?Qatar",
        html,
        flags=re.IGNORECASE | re.DOTALL
    )

    prices = []

    for match in qatar_sections:
        for p in match:
            if p:
                try:
                    value = int(p.replace(",", ""))
                    if 300 <= value <= 5000:
                        prices.append(value)
                except:
                    pass

    if not prices:
        print("No Qatar fare price detected.")
        return None

    return min(prices)

# ==========================
# MAIN
# ==========================

def main():

    old_prices = load_prices()
    new_prices = {}

    options = webdriver.ChromeOptions()

    # Remove this line if you want to see Chrome open
    options.add_argument("--headless=new")

    options.add_argument("--disable-blink-features=AutomationControlled")
    options.add_argument("--window-size=1920,1080")

    driver = webdriver.Chrome(
        service=Service(ChromeDriverManager().install()),
        options=options
    )

    try:
        price = get_qatar_price(driver)

        if price is None:
            send_notification(
                "Qatar Flight Tracker",
                "Could not find Qatar fare for MSY → KTM."
            )
            return

        print(f"Qatar fare found: ${price}")

        new_prices["QATAR_MSY_KTM"] = price
        previous = old_prices.get("QATAR_MSY_KTM")

        if previous is None:
            send_notification(
                "Qatar Flight Tracker",
                f"Baseline saved: MSY → KTM Qatar fare ${price}"
            )

        else:
            change = price - previous

            if change <= -ALERT_THRESHOLD:
                send_notification(
                    "✈ Qatar Fare Dropped",
                    f"MSY → KTM\n${previous} → ${price}\nSaved ${abs(change)}"
                )

            elif change >= ALERT_THRESHOLD:
                send_notification(
                    "⚠ Qatar Fare Increased",
                    f"MSY → KTM\n${previous} → ${price}\n+${change}"
                )

        if price <= TARGET_PRICE:
            send_notification(
                "🔥 Qatar Kathmandu Deal",
                f"MSY → KTM Qatar fare is ${price}"
            )

    finally:
        driver.quit()

    save_prices(new_prices)
    print("Done.")

# ==========================
# RUN
# ==========================

if __name__ == "__main__":
    main()
