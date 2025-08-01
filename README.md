import speech_recognition as sr
import webbrowser
import pyttsx3
import pywhatkit
import tkinter as tk
import threading
from PIL import Image, ImageTk
import openai
import sounddevice as sd
import numpy as np

# —– Voice engine setup —–
recognizer = sr.Recognizer()
engine = pyttsx3.init()
openai.api_key = "YOUR_OPENAI_KEY"

def speak(text):
    engine.say(text)
    engine.runAndWait()

def ask_chatgpt(prompt):
    client = openai.Client("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")
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": prompt}]
    )
    reply = response['choices'][0]['message']['content']
    speak(reply)

def processcommand(c):
    cmd = c.lower()
    if cmd.startswith("open "):
        site = cmd.replace("open ", "").strip()
        webbrowser.open(f"https://{site}")
    elif cmd.startswith("play"):
        song = cmd.replace("play", "").strip()
        pywhatkit.playonyt(song)
    elif "chatgpt" in cmd:
        q = cmd.replace("chatgpt", "").strip()
        if q:
            speak("Asking ChatGPT...")
            ask_chatgpt(q)
        else:
            speak("Please say something to ask ChatGPT.")
    elif "exit" in cmd or "shutdown" in cmd:
        speak("Shutting down. Goodbye!")
        root.quit()
    else:
        speak("Sorry, I didn't understand that.")

 #—– GUI setup —–
root = tk.Tk()
root.title("Jarvis Voice Assistant")
root.geometry("500x600")
root.configure(bg="#111111")
root.resizable(False, False)

glass = tk.Frame(root, bg="#1e1e1e", highlightthickness=1, highlightbackground="#00ffff")
glass.place(relx=0.5, rely=0.5, anchor="center", width=400, height=500)

# Icon (optional)
try:
    img = Image.open("jarvis_icon.png").resize((120, 120), Image.ANTIALIAS)
    logo = ImageTk.PhotoImage(img)
    lbl = tk.Label(glass, image=logo, bg="#1e1e1e")
    lbl.pack(pady=20)
except:
    pass

tk.Label(glass, text="JARVIS", font=("Helvetica", 28, "bold"), fg="#00ffff", bg="#1e1e1e").pack()
subtitle = tk.Label(glass, font=("Helvetica", 14), fg="#cccccc", bg="#1e1e1e")
subtitle.pack(pady=10)

def type_effect(label, text, delay=100):
    label.config(text="")
    def anim(i=0):
        if i <= len(text):
            label.config(text=text[:i])
            label.after(delay, anim, i+1)
    anim()

type_effect(subtitle, "Your Personal Voice Assistant")

listen_button = tk.Button(
    glass, text="🎙️ Listen", font=("Helvetica", 18, "bold"),
    bg="#00ffff", fg="#000", activebackground="#00cccc", relief="flat",
    height=2, width=12, cursor="hand2",
    command=lambda: threading.Thread(target=listen_flow, daemon=True).start()
)
listen_button.pack(pady=20)

tk.Label(glass, text="Made by Ritesh", font=("Helvetica", 10), fg="gray", bg="#1e1e1e").pack(side="bottom", pady=10)

# —– Waveform canvas —–
mic_canvas = tk.Canvas(glass, width=300, height=100, bg="#1e1e1e", highlightthickness=0)
mic_canvas.pack(pady=10)
wave_line = mic_canvas.create_line(0, 50, 300, 50, fill="#00ffff")

def animate_glow(step=0):
    colors = ["#00ffff", "#00cccc"]
    listen_button.config(bg=colors[step])
    next_step = 1 - step
    if listen_button["text"].startswith("🎤"):
        listen_button.after(500, animate_glow, next_step)

def stream_waveform():
    def callback(indata, frames, time, status):
        data = indata[:, 0]
        wave = ((data / (np.max(np.abs(data)) + 1e-6)) * 45) + 50
        coords, step = [], 300 / len(wave)
        for i, y in enumerate(wave):
            coords += [i * step, y]
        mic_canvas.coords(wave_line, *coords)

    with sd.InputStream(callback=callback, channels=1, samplerate=44100, blocksize=1024):
        while listen_button["text"].startswith("🎤"):
            mic_canvas.update()

# —– Listening flow combining glow + waveform + recognition —–
def listen_flow():
    try:
        listen_button.config(text="🎤 Listening...", bg="#ffaa00")
        animate_glow()
        threading.Thread(target=stream_waveform, daemon=True).start()

        with sr.Microphone() as src:
            recognizer.adjust_for_ambient_noise(src, duration=0.5)
            speak("Listening for wake word...")
            audio = recognizer.listen(src, timeout=5, phrase_time_limit=5)
            words = recognizer.recognize_google(audio)
        if "jarvis" in words.lower():
            speak("Hello sir, I'm listening.")
            with sr.Microphone() as src:
                recognizer.adjust_for_ambient_noise(src, duration=0.5)
                speak("What can I do for you?")
                audio = recognizer.listen(src, timeout=6, phrase_time_limit=6)
                cmd = recognizer.recognize_google(audio)
                processcommand(cmd)
        else:
            speak("Wake word not detected.")
    except Exception as e:
        print("Error:", e)
        speak("Something went wrong.")
    finally:
        listen_button.config(text="🎙️ Listen", bg="#00ffff")

# Start everything
speak("Initializing Jarvis...")
root.mainloop()
