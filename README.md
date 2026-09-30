import ctypes
from ctypes import wintypes
import threading
import time
import psutil
import tkinter as tk

# --- Windows API Constants & Structs ---
WH_KEYBOARD_LL = 13
WM_KEYDOWN = 0x0100

class KBDLLHOOKSTRUCT(ctypes.Structure):
    _fields_ = [
        ("vkCode", wintypes.DWORD),
        ("scanCode", wintypes.DWORD),
        ("flags", wintypes.DWORD),
        ("time", wintypes.DWORD),
        ("dwExtraInfo", ctypes.c_ulonglong)
    ]

# Whitelist of trusted system applications that normally use hooks/inputs
TRUSTED_PROCESSES = ["explorer.exe", "textinputhost.exe", "ctfmon.exe", "python.exe"]

class KeyloggerDetectorApp:
    def __init__(self):
        # GUI Setup (Overlay Window)
        self.root = tk.Tk()
        self.root.title("Keylogger Alert")
        self.root.geometry("400x100+50+50") # Width x Height + X_offset + Y_offset
        self.root.attributes("-topmost", True) # Hamesha Screen ke Upar
        self.root.overrideredirect(True) # Title bar remove karne ke liye
        self.root.configure(bg="#1E1E1E")
        
        # UI Elements
        self.status_label = tk.Label(
            self.root, 
            text="🛡️ KeySentinel Active - Monitoring...", 
            font=("Segoe UI", 11, "bold"), 
            fg="#00FF7F", 
            bg="#1E1E1E"
        )
        self.status_label.pack(pady=10)
        
        self.detail_label = tk.Label(
            self.root, 
            text="No suspicious hooks detected.", 
            font=("Segoe UI", 9), 
            fg="#CCCCCC", 
            bg="#1E1E1E"
        )
        self.detail_label.pack()

        self.kill_btn = tk.Button(
            self.root, 
            text="Terminate Process", 
            command=self.kill_flagged_process, 
            bg="#FF3333", 
            fg="white", 
            font=("Segoe UI", 9, "bold"),
            state=tk.DISABLED
        )
        self.kill_btn.pack(pady=5)

        self.flagged_pid = None

        # Start Background Hook Monitor Thread
        self.monitor_thread = threading.Thread(target=self.start_hook_monitor, daemon=True)
        self.monitor_thread.start()

    def update_ui_alert(self, process_name, pid):
        """Threat detect hone par Screen Banner update karne ke liye"""
        self.flagged_pid = pid
        self.root.configure(bg="#721C24")
        self.status_label.configure(
            text=f"⚠️ KEYLOGGER DETECTED: {process_name}", 
            fg="#FFD700", 
            bg="#721C24"
        )
        self.detail_label.configure(
            text=f"Process PID: {pid} is actively capturing keystrokes!", 
            fg="#FFFFFF", 
            bg="#721C24"
        )
        self.kill_btn.configure(state=tk.NORMAL)

    def kill_flagged_process(self):
        """User alert button click karke process terminate kar sakta hai"""
        if self.flagged_pid:
            try:
                p = psutil.Process(self.flagged_pid)
                p.terminate()
                self.status_label.configure(text="✅ Threat Terminated Successfully!", fg="#00FF7F")
                self.kill_btn.configure(state=tk.DISABLED)
                self.flagged_pid = None
            except Exception as e:
                self.detail_label.configure(text=f"Error killing process: {str(e)}")

    def start_hook_monitor(self):
        """Windows low-level hook callback handler"""
        user32 = ctypes.windll.user32
        
        # LowLevelKeyboardProc Callback definition
        HOOKPROC = ctypes.WINFUNCTYPE(ctypes.c_int, ctypes.c_int, wintypes.WPARAM, wintypes.LPARAM)
        
        def hook_callback(nCode, wParam, lParam):
            if nCode >= 0 and wParam == WM_KEYDOWN:
                # Active Window/Process extract karna jo key read kar raha ha
                hwnd = user32.GetForegroundWindow()
                pid = wintypes.DWORD()
                user32.GetWindowThreadProcessId(hwnd, ctypes.byref(pid))
                
                try:
                    process = psutil.Process(pid.value)
                    proc_name = process.name().lower()
                    
                    # Agar process trusted list mein nahi hai toh banner alert show karein
                    if proc_name not in TRUSTED_PROCESSES:
                        self.root.after(0, self.update_ui_alert, proc_name, pid.value)
                except (psutil.NoSuchProcess, psutil.AccessDenied):
                    pass
                
            return user32.CallNextHookEx(None, nCode, wParam, lParam)

        callback_func = HOOKPROC(hook_callback)
        hook_id = user32.SetWindowsHookExA(WH_KEYBOARD_LL, callback_func, None, 0)

        # Windows Message Loop (Hook events process karne ke liye)
        msg = wintypes.MSG()
        while user32.GetMessageA(ctypes.byref(msg), None, 0, 0) != 0:
            user32.TranslateMessage(ctypes.byref(msg))
            user32.DispatchMessageA(ctypes.byref(msg))

        user32.UnhookWindowsHookEx(hook_id)

    def run(self):
        self.root.mainloop()

if __name__ == "__main__":
    app = KeyloggerDetectorApp()
    app.run()
