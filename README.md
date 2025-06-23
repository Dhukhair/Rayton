# === Rayton SuperApp v2.1: Интерактивное Символическое IDE + ИСКИН ===
# Полная версия с учетом отсутствия pyaudio: безопасное выполнение голоса
# Добавлено: интуитивное автообучение на любом чате (вне зависимости от команды)

import tkinter as tk
from tkinter import filedialog, messagebox, scrolledtext
import json, pyperclip, pyttsx3, os, time, threading, random
import speech_recognition as sr

memory_file = "memory.json"
session_file = "last_session.rayproj"
chat_history = []
memory = {}
sessions = []
current_session_index = 0
engine = pyttsx3.init()
variables = {}

# --- Загрузка/сохранение памяти ---
def load_memory():
    global memory
    try:
        with open(memory_file, "r", encoding="utf-8") as f:
            memory = json.load(f)
    except:
        memory = {}

def save_memory():
    with open(memory_file, "w", encoding="utf-8") as f:
        json.dump(memory, f, ensure_ascii=False, indent=2)

def remember(key, value):
    memory[key] = value
    save_memory()

def recall(key):
    return memory.get(key, "")

# --- Импорт знаний из JSON ---
def import_knowledge_from_json(file_path):
    try:
        with open(file_path, "r", encoding="utf-8") as f:
            data = json.load(f)
        count = 0
        for key, values in data.items():
            if key in memory:
                for v in values:
                    if v not in memory[key]:
                        memory[key].append(v)
                        count += 1
            else:
                memory[key] = values
                count += len(values)
        save_memory()
        return f"📥 Импортировано {count} новых знаний."
    except Exception as e:
        return f"⚠️ Ошибка при импорте: {e}"

# --- Raython интерпретатор ---
def run_raython(code_lines, output_callback):
    global variables
    variables = {}
    for line in code_lines:
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        if line.startswith("ا"):
            pass
        elif line.startswith("ب"):
            parts = line.split(None, 2)
            if len(parts) == 3:
                val = parts[2].strip('"')
                variables[parts[1]] = val
        elif line.startswith("د"):
            parts = line.split(None, 1)
            val = parts[1].strip('"')
            output_callback(val)
        elif line.startswith("ذ"):
            parts = line.split(None, 1)
            val = parts[1].strip('"')
            output_callback("[Вывод]: " + val)
        elif line.startswith("запомнить"):
            parts = line.split(None, 1)
            remember(parts[1], variables.get(parts[1], ""))
        elif line.startswith("вспомнить"):
            parts = line.split(None, 1)
            variables[parts[1]] = recall(parts[1])
        elif line.startswith("س"):
            cond = line[1:].strip()
            try:
                cond_eval = eval(cond, {}, variables)
                if not cond_eval:
                    return
            except:
                return
        elif line.startswith("خ"):
            continue
        elif line.startswith("ع") and "запомни" in line:
            keyval = line.split("запомни", 1)[1].strip()
            if "," in keyval:
                k, v = keyval.split(",", 1)
                remember(k.strip(), v.strip())

# --- Чат и обучение ---
def chat_response(user_msg):
    user_msg = user_msg.lower().strip()

    if user_msg.startswith("запомни:"):
        try:
            raw = user_msg.replace("запомни:", "").strip()
            if "->" in raw:
                key, val = map(str.strip, raw.split("->", 1))
                memory.setdefault(key, [])
                if val not in memory[key]:
                    memory[key].append(val)
                save_memory()
                return f"Запомнил, что '{key}' → '{val}'"
            else:
                return "❗ Формат: запомни: ключ -> значение"
        except Exception as e:
            return f"⚠️ Ошибка: {e}"

    for key in memory:
        if key in user_msg:
            return random.choice(memory[key])

    if user_msg == "запомни это":
        if len(chat_history) >= 2:
            q = chat_history[-2].get("user", "").strip().lower()
            a = chat_history[-1].get("iskin", "").strip()
            memory.setdefault(q, [])
            if a not in memory[q]:
                memory[q].append(a)
                save_memory()
                return "🧠 Запомнил из чата."
        return "⚠️ Нет данных."

    if "время" in user_msg:
        return time.strftime("Сейчас %H:%M:%S")

    return "🤖 Я этого пока не знаю. Скажи 'запомни: вопрос -> ответ'."

def auto_learn_from_history():
    pairs = []
    for i in range(len(chat_history)-1):
        q = chat_history[i].get("user", "").strip().lower()
        a = chat_history[i+1].get("iskin", "").strip()
        if not a or "я этого пока не знаю" in a:
            continue
        memory.setdefault(q, [])
        if a not in memory[q]:
            memory[q].append(a)
            pairs.append((q, a))
    if pairs:
        save_memory()
        return f"Обучено на {len(pairs)} фраз."
    return "Нет новых фраз."

def voice_input(callback):
    try:
        r = sr.Recognizer()
        with sr.Microphone() as source:
            audio = r.listen(source)
            text = r.recognize_google(audio, language="ru-RU")
            callback(text)
    except:
        callback("[Голосовой ввод не доступен]")

def save_session(code_text):
    session = {
        "code": code_text,
        "memory": memory,
        "chat": chat_history
    }
    if current_session_index < len(sessions):
        sessions[current_session_index] = session
    else:
        sessions.append(session)
    with open(session_file, "w", encoding="utf-8") as f:
        json.dump(sessions, f, ensure_ascii=False, indent=2)

def load_session():
    global sessions
    if os.path.exists(session_file):
        with open(session_file, "r", encoding="utf-8") as f:
            sessions = json.load(f)
        return sessions[0] if sessions else {"code": "", "memory": {}, "chat": []}
    return {"code": "", "memory": {}, "chat": []}

def build_gui():
    global current_session_index
    load_memory()
    session = load_session()
    memory.update(session.get("memory", {}))
    chat_history.extend(session.get("chat", []))

    root = tk.Tk()
    root.title("Rayton SuperApp v2.1 — Символическая среда ИСКИН")

    code_box = scrolledtext.ScrolledText(root, width=80, height=20, font=("Consolas", 12))
    code_box.pack(padx=10, pady=5)
    code_box.insert(tk.END, session.get("code", "") or example_code())

    output_box = scrolledtext.ScrolledText(root, width=80, height=10, font=("Consolas", 11), bg="#f2f2f2")
    output_box.pack(padx=10, pady=5)

    chat_frame = tk.Frame(root)
    chat_frame.pack()
    chat_entry = tk.Entry(chat_frame, width=60)
    chat_entry.grid(row=0, column=0)

    def process_chat(msg):
        if msg:
            chat_history.append({"user": msg})
            res = chat_response(msg)
            chat_history.append({"iskin": res})
            output_box.insert(tk.END, f"🧑: {msg}\n🤖: {res}\n")
            try:
                engine.say(res)
                engine.runAndWait()
            except:
                pass
            chat_entry.delete(0, tk.END)

    tk.Button(chat_frame, text="Отправить", command=lambda: process_chat(chat_entry.get())).grid(row=0, column=1)
    tk.Button(chat_frame, text="🎙️Голос", command=lambda: threading.Thread(target=voice_input, args=(process_chat,)).start()).grid(row=0, column=2)

    btns = tk.Frame(root)
    btns.pack(pady=5)

    def run_code():
        output_box.delete("1.0", tk.END)
        lines = code_box.get("1.0", tk.END).split("\n")
        run_raython(lines, lambda msg: output_box.insert(tk.END, msg + "\n"))

    tk.Button(btns, text="▶ Выполнить", command=run_code).grid(row=0, column=0)
    tk.Button(btns, text="💾 Сохранить", command=lambda: save_session(code_box.get("1.0", tk.END))).grid(row=0, column=1)
    tk.Button(btns, text="📋 Копировать", command=lambda: pyperclip.copy(code_box.get("1.0", tk.END))).grid(row=0, column=2)
    tk.Button(btns, text="📥 Вставить", command=lambda: code_box.insert(tk.INSERT, pyperclip.paste())).grid(row=0, column=3)
    tk.Button(btns, text="💡 Запомнить", command=lambda: remember("выделение", code_box.get(tk.SEL_FIRST, tk.SEL_LAST))).grid(row=0, column=4)
    tk.Button(btns, text="🔁 Вспомнить", command=lambda: code_box.insert(tk.INSERT, recall("выделение"))).grid(row=0, column=5)
    tk.Button(btns, text="📚 Автообучение", command=lambda: output_box.insert(tk.END, auto_learn_from_history() + "\n")).grid(row=0, column=6)
    tk.Button(btns, text="❌ Выход", command=lambda: (save_session(code_box.get("1.0", tk.END)), root.destroy())).grid(row=0, column=7)

    root.mainloop()

def example_code():
    return '''ا имя
ب имя "Азамат"
د имя

ا вопрос
ب вопрос "Как тебя зовут?"
د вопрос

س вопрос == "Как тебя зовут؟"
  ب ответ "Меня зовут " + имя
  د ответ
خ'''

if __name__ == "__main__":
    build_gui()
