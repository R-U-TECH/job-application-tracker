# job-application-tracker
import subprocess
import sys
import time

# Automatically check and install missing libraries
for module_name, pip_name in [("imap_tools", "imap-tools"), ("pandas", "pandas"), ("openpyxl", "openpyxl")]:
    try:
        __import__(module_name)
    except ImportError:
        print(f"Installing required library: {pip_name}...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", pip_name])

from imap_tools import MailBox, AND
import pandas as pd
import datetime
import os

# --- USER CREDENTIALS ---
# WARNING: Do NOT use your regular email password. 
# You MUST generate a 16-character App Password via Google/Microsoft Security settings.
EMAIL = "yourmail@gmail.com"
PASSWORD = "your_app_password"
IMAP_SERVER = "imap.gmail.com"


def scan_only_applications():
    application_list = []
    
    # Historical boundary set to January 1, 2022
    date_limit = datetime.date(2022, 1, 1)
    
    # Safe Swedish character escape sequences to prevent compiler encoding errors
    swedish_ansokan = "ans\u00f6kan"
    swedish_bekraftelse = "bekr\u00e4ftelse"
    
    # Multilingual tracking keywords (English, Swedish, Turkish)
    keywords = [
        "application received", "thank you for applying", "thank you for your application",
        "successfully submitted", "your application to", "application confirmation",
        "mottagit din", swedish_ansokan, swedish_bekraftelse, "tack",
        "basvurunuz", "basvuru onayi", "basvurunuzu aldik", "is basvurusu"
    ]
    
    print("Connecting to secure mailbox (Applications Mode: 2022 - Present)...")
    try:
        with MailBox(IMAP_SERVER, timeout=45).login(EMAIL, PASSWORD, initial_folder='INBOX') as mailbox:
            print("Connected successfully! Streaming ONLY application confirmations...")
            
            # Fetching only headers to dramatically boost speed and avoid rate limits
            for msg in mailbox.fetch(AND(date_gte=date_limit), reverse=True, headers_only=True):
                try:
                    subject = msg.subject if msg.subject else ""
                    sender = msg.from_ if msg.from_ else ""
                    
                    # Convert email date object to global UTC format for accuracy
                    if msg.date:
                        utc_date = msg.date.astimezone(datetime.timezone.utc)
                        date_str = utc_date.strftime('%d.%m.%Y %H:%M') + " UTC"
                    else:
                        date_str = ""
                    
                    if any(kw in subject.lower() for kw in keywords):
                        print(f"[SECURED] {date_str} -> {subject}")
                        application_list.append({
                            "Date_UTC": date_str,
                            "Company_Sender": sender,
                            "Position_Subject": subject
                        })
                    
                    # Small human-like pacing delay to protect connection from getting banned
                    time.sleep(0.08)
                except Exception:
                    continue
    except Exception as network_err:
        print(f"\n[Connection lost] Saving what we safely captured so far... Error: {network_err}")
        
    return application_list

if __name__ == "__main__":
    try:
        data = scan_only_applications()
        if data:
            df = pd.DataFrame(data)
            
            # Get the path to user's Desktop
            desktop_path = os.path.join(os.path.join(os.environ['USERPROFILE']), 'Desktop')
            file_path = os.path.join(desktop_path, "My_Applications_History_2022_Present.xlsx")
            
            df.to_excel(file_path, index=False, engine="openpyxl")
            print(f"\n VICTORY! Your pure application history is saved to your DESKTOP as 'My_Applications_History_2022_Present.xlsx'")
            print(f"Total entries captured: {len(data)}")
        else:
            print("\nScan completed. No application entries found.")
    except Exception as e:
        print(f"\nAn error occurred during file save: {e}")
